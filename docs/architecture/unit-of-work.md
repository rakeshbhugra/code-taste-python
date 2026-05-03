
# Unit of Work (UoW)

← [Back to index](./README.md)

## One-line definition

A Unit of Work is **the object that owns a transaction**: it groups a bunch of in-memory domain changes into one atomic commit-or-rollback.

If Repository is "the seam to storage," Unit of Work is "the *transaction boundary* around using that seam." It's how you say *"these three changes either all happen or none of them do."*

## The problem it solves

You have a service that touches multiple things:

```python
def share_list(list_id, owner_id, recipient_email, lists, users, notifications):
    lst = lists.get(list_id)
    recipient = users.get_by_email(recipient_email)

    lst.share_with(recipient.id)
    recipient.add_shared_list(list_id)

    lists.add(lst)
    users.add(recipient)
    notifications.send(...)  # ← what if this fails?
```

Without a UoW, you're in trouble:

1. **Partial writes.** If `users.add(recipient)` succeeds but the connection drops before `notifications.send()`, the DB has the share recorded but no email goes out. Silent inconsistency.
2. **Who calls `commit()`?** The repositories? Each one? After every method? You either commit too eagerly (and can't roll back as a group) or scatter `commit()` calls across the service (and forget one).
3. **Tests can't reset state cleanly.** Without a transaction wrapping the whole operation, undoing test side-effects requires bespoke teardown for each test.

The UoW gives you **one explicit "begin → do stuff → commit-or-rollback" boundary** that wraps an entire use case.

## What it looks like

A UoW is almost always a **context manager**. The pattern:

```python
class UnitOfWork(ABC):
    lists: ListRepository
    users: UserRepository

    def __enter__(self) -> "UnitOfWork":
        return self

    def __exit__(self, exc_type, *args):
        if exc_type is None:
            # No exception — but DON'T auto-commit. Caller must commit explicitly.
            self.rollback()
        else:
            self.rollback()

    @abstractmethod
    def commit(self): ...

    @abstractmethod
    def rollback(self): ...
```

And in the service:

```python
def share_list(list_id, owner_id, recipient_email, uow: UnitOfWork) -> None:
    with uow:
        lst = uow.lists.get(list_id)
        recipient = uow.users.get_by_email(recipient_email)

        lst.share_with(recipient.id)
        recipient.add_shared_list(list_id)

        uow.lists.add(lst)
        uow.users.add(recipient)

        uow.commit()
```

Three things to notice:

1. **Repositories now hang off the UoW.** `uow.lists`, `uow.users` — not separate injected dependencies. The UoW *owns* its repos because they all share the same transaction.
2. **The service explicitly calls `uow.commit()`.** If it doesn't, `__exit__` rolls back. The book's argument: *"explicit is better than implicit"* — you should be able to look at a service and see exactly when the transaction commits.
3. **The service no longer takes a `repo` argument.** It takes the UoW. Everything DB-related goes through the UoW.

## Why context managers

Two reasons it's `with uow:` instead of plain function calls:

1. **Cleanup is guaranteed.** If your service raises halfway through, `__exit__` runs and rolls back. No leaked DB connections, no half-applied changes. Same guarantee a `try/finally` gives you, but cheaper to write.
2. **The boundary is visually obvious.** You can read a service and instantly see: *here's where the transaction starts, here's where it ends, the commit is right there.* Without `with`, you'd be reading 30 lines of code asking "are we in a transaction or not?"

This is also why SQLAlchemy's `session` is essentially a UoW — it's already a context-manager-shaped transaction-owner. The UoW pattern is mostly a thin layer that gives you a **swappable** SQLAlchemy session (so you can replace it with a fake in tests).

## Real examples — mapped to your projects

| Use case | Why it needs a UoW |
|---|---|
| Hypedar: share a list | Updates `List` *and* `User` (recipient gains the share). Both succeed or neither. |
| Hypedar: upgrade to pro | Charges payment *and* updates `User.plan` *and* records `payment_id`. Atomicity is critical — billing without plan upgrade is a support ticket. |
| Tytona: send a message | Updates `Conversation` *and* increments user's monthly counter. From Q3 in service questions — the load → work → save shape needs to be one transaction. |
| Hypedar: create a list | Single entity, single repo. *Probably doesn't need a UoW* — but using one keeps the pattern consistent across services. |

The pattern: **if a service touches more than one repository OR you care about all-or-nothing semantics, you want a UoW.** Even single-repo cases benefit from the consistency.

## What's in / what's out

**In the UoW:**
- The transaction (begin / commit / rollback)
- References to all the repositories that share the transaction
- Lifecycle (`__enter__`, `__exit__`)

**Out of the UoW:**
- Business rules (those still live in the domain)
- The actual DB queries (those still live in the repositories — UoW doesn't know SQL)
- Use case orchestration (the service still drives the flow; UoW just provides the transactional environment)

## The traps

### Trap 1: forgetting to commit

```python
def share_list(list_id, recipient_email, uow):
    with uow:
        lst = uow.lists.get(list_id)
        lst.share_with(recipient_email)
        uow.lists.add(lst)
        # ← forgot uow.commit()
```

`__exit__` runs with no exception, doesn't see a commit, rolls back. The user clicks "Share," gets a 200 OK, but nothing happened in the DB. Worst kind of bug — silent failure.

The book's defense: making commit *explicit* means missing it is loud. The first integration test will catch it. Implicit-commit-on-exit feels safer but masks bugs (commits happen even when the service didn't intend them to).

### Trap 2: nested or long-running transactions

```python
def fancy_workflow(uow):
    with uow:
        lst = uow.lists.get(...)
        result = call_openai_api(lst)   # ← 5-second network call inside the transaction!
        lst.summary = result
        uow.commit()
```

Holding a DB transaction open for 5 seconds while waiting on OpenAI is a nightmare for concurrency. The fix: do external I/O *before* opening the transaction, or split it:

```python
def fancy_workflow(uow):
    with uow:
        lst = uow.lists.get(...)
    # transaction released, expensive call outside
    result = call_openai_api(lst)
    with uow:
        lst = uow.lists.get(...)        # re-fetch, with version check ideally
        lst.summary = result
        uow.commit()
```

Rule of thumb: **transactions should be short and CPU/DB-bound, never network-bound to external services.**

### Trap 3: leaking the UoW above the service

```python
@router.post("/lists")
def create_list_route(req, uow = Depends(get_uow)):
    with uow:                          # ← route managing the transaction
        services.create_list(...)
        uow.commit()
```

The route shouldn't know about transactions. The service should own the `with uow:` block. Otherwise the route is doing infrastructure work, and you can't reuse the service from a worker without rewriting the transaction handling.

The right shape:

```python
@router.post("/lists")
def create_list_route(req, uow = Depends(get_uow)):
    return services.create_list(req, uow)   # ← service handles the transaction internally

# In the service:
def create_list(req, uow):
    with uow:
        ...
        uow.commit()
```

### Trap 4: doing too much in one transaction

A UoW wraps **one logical operation**. If your service is doing five unrelated things at once, you've got a use-case design problem, not a UoW problem. Smaller, focused services with smaller transactions beat one mega-service with a huge transaction.

### Trap 5: thinking you need a UoW when SQLAlchemy already has a session

SQLAlchemy's `Session` is essentially already a UoW — it's a context-managed transaction with an identity map. If you're not planning to swap SQLAlchemy and you don't need fake-out-for-tests, you can use `session.begin()` directly and skip the explicit UoW class. The UoW abstraction earns rent when you want a `FakeUnitOfWork` for tests, or you might swap your data layer.

## How to spot it in your code

In hypedar today, look at any service that updates multiple things. The smell:

```python
def share_list(...):
    lst = self._repo.get(...)
    user = self._repo.get_user(...)
    lst.shared_with.append(user.id)
    self._repo.save(lst)               # commits here?
    self._repo.save_user(user)         # commits again?
    # If save_user fails, lst is already saved. Inconsistent.
```

Three smells in one:
1. Each `save_*` call probably commits independently (no atomicity)
2. There's no explicit transaction boundary — *when* are we committing?
3. A failure halfway through leaves the DB in a half-updated state

The cleanup with a UoW:

```python
def share_list(self, list_id, recipient_email, uow):
    with uow:
        lst = uow.lists.get(list_id)
        user = uow.users.get_by_email(recipient_email)
        lst.share_with(user.id)
        user.add_shared_list(list_id)
        uow.lists.add(lst)
        uow.users.add(user)
        uow.commit()
```

Both saves are now part of the same transaction. The boundaries are visible. A failure rolls everything back. This is the difference between "code that works most of the time" and "code that's correct under partial-failure conditions."

## The hidden benefit

The big one is atomicity, which we've covered. But two subtler ones:

1. **Tests get a free reset.** A `FakeUnitOfWork` can be a context manager whose `__exit__` clears its state. Every test is a fresh transaction. No DB cleanup, no fixture teardown.

```python
class FakeUnitOfWork(UnitOfWork):
    def __init__(self):
        self.lists = InMemoryListRepository()
        self.users = InMemoryUserRepository()
        self.committed = False

    def commit(self): self.committed = True
    def rollback(self): pass

# Tests:
def test_share_list_commits_when_successful():
    uow = FakeUnitOfWork()
    services.share_list("list1", "alice@example.com", uow)
    assert uow.committed
```

2. **You can verify "did this commit?" in tests.** With the FakeUoW you can directly assert that `commit()` was called (or not). That makes it possible to test failure paths cleanly: simulate an exception mid-service, assert that `commit` was not called and the in-memory state was rolled back.

This is what the book means by *"explicit commits make tests sharper."*

## Self-check

Try answering in your own words:

1. What does a UoW give you that a Repository alone doesn't?
2. Why does the UoW typically use a `with` block instead of explicit `begin()`/`end()` calls?
3. The book argues commits should be explicit (`uow.commit()`) rather than auto-on-exit. Why?
4. A service does one repo call to fetch a `List`, mutates it, saves it. Does it need a UoW? What's the trade-off?
5. Why is calling an LLM API inside `with uow:` a smell?

If those land, you've got Unit of Work. **You've now finished the core five.**

## Things to steal

- **One UoW = one logical operation.** Not one HTTP request, not one entity, not one SQL statement. One business-level "thing happens."
- **Explicit commits.** Don't rely on auto-commit-on-exit. Make the commit visible in the service code.
- **Repositories hang off the UoW.** They share a transaction; group them under one owner.
- **Keep transactions short.** No external API calls inside `with uow:`. Do I/O before or after, never during.
- **The service owns the transaction, not the route.** Routes are transport; transactions are use-case-shaped.
- **`FakeUnitOfWork` is the killer test feature.** Same lesson as `FakeRepository` — the abstraction earns most of its rent in tests.
