
# Testing

← [Back to index](./README.md)

Testing has its own vocabulary and a small set of load-bearing concepts. This doc is the orientation — the rest you learn by doing.

## The goal (the only one)

> **Tests give you confidence to change code without breaking things.**

Everything else is a means to that end. Coverage %, TDD purity, "100 tests look impressive" — none of it matters if the tests don't make the codebase safer to change. Conversely, if you can refactor confidently, you have enough tests, even at 41% coverage (orqestrator's current ratchet point).

The architecture you just learned exists *because* it makes tests cheap. That's the connection — pure domain → fast unit tests; ports/adapters → cheap fakes; UoW → tests reset for free.

## The test pyramid (the central concept)

```
                  ╱──────╲
                 ╱  e2e   ╲          ← few. slow. flaky-prone. test the wires.
                ╱──────────╲
               ╱ integration╲        ← some. medium speed. test adapters
              ╱──────────────╲          and seams.
             ╱      unit      ╲      ← many. fast. test domain rules
            ╱──────────────────╲       and service orchestration.
```

**Read it as: the further you go from your domain, the slower and fewer the tests.**

| Type | What it tests | Speed | Count | Example |
|---|---|---|---|---|
| **Unit** | One domain class or pure function | <10 ms | hundreds | `test_list_add_item_raises_when_full` |
| **Service / contract** | One service, with fake adapters | tens of ms | tens to low hundreds | `test_create_list_assigns_slug` |
| **Integration** | Real adapter against real external system | hundreds of ms | tens | `test_sqlalchemy_list_repo_can_save_and_retrieve` |
| **E2E** | Full HTTP request through every layer | seconds | low single digits per critical flow | `test_post_lists_creates_a_list` |

The pyramid is **lots of unit, fewer integration, very few e2e.** The inverse pyramid (lots of e2e, few unit) is a smell — it means your business logic isn't testable in isolation, which means it's not actually in the domain.

## The three test types in detail

### Unit tests — the workhorse

Test **one domain class or pure function** with **no I/O**. No DB, no HTTP, no filesystem, no network, no time, no randomness (or all controlled).

```python
def test_list_add_item_raises_when_full():
    lst = List(id="l1", name="x", items=[Item(id=str(i)) for i in range(100)])
    with pytest.raises(ListFull):
        lst.add_item(Item(id="new"))
```

If your unit tests need to mock `boto3` or `litellm`, **the code being tested isn't actually domain** — infrastructure has leaked in. The test failure is telling you about an architecture problem, not a test problem.

### Service tests — using `FakeUnitOfWork` / `FakeRepository`

Test **one service function** with **fake adapters in place of real ones**. This is where the architecture pays off — you can test "create_list assigns a slug and persists" without spinning up a database.

```python
def test_create_list_persists_with_slug():
    uow = FakeUnitOfWork()
    services.create_list(user_id="u1", name="My Stuff", uow=uow)
    saved = uow.lists.get_by_user("u1")[0]
    assert saved.slug == "my-stuff"
    assert uow.committed
```

These run in milliseconds because `FakeUnitOfWork` is just a dict in memory. You can have hundreds of these and the suite still finishes in under a second.

### Integration tests — real adapter, real external system

Test **the adapter itself**, against the real thing. This is the **only** place you need a real DB / real network.

```python
def test_sqlalchemy_list_repo_round_trip(session):
    repo = SqlAlchemyListRepository(session)
    lst = List.create(name="Test")
    repo.add(lst)
    session.commit()

    retrieved = repo.get(lst.id)
    assert retrieved.name == "Test"
```

You write **few** of these — maybe one per adapter method that has non-trivial mapping logic. The point is to catch ORM/SQL mistakes, not to retest the domain.

### E2E tests — the full stack

Test **the actual HTTP flow** with a real or near-real environment. FastAPI's `TestClient` or `httpx` against a running app.

```python
def test_create_list_endpoint(client):
    response = client.post("/lists", json={"name": "My Stuff"})
    assert response.status_code == 201
    assert response.json()["slug"] == "my-stuff"
```

Write **very few** of these — one or two per critical user flow (signup, payment, share). They're slow, brittle, and hard to debug. Their job is to catch wiring mistakes, not logic mistakes.

## Test doubles — the four words people confuse

Often used interchangeably; they mean different things.

| Term | What it does | When to use |
|---|---|---|
| **Fake** | A working implementation, just simpler (in-memory instead of DB) | The default. `InMemoryListRepository`, `FakeNotifier`. Behaves like the real thing. |
| **Stub** | Returns canned values, no behavior | Quick one-off when you only need a return value |
| **Mock** | Records calls so you can assert on them. Pre-programmed expectations. | Asserting *that something was called* — `assert mock_emailer.send.called_once_with(...)` |
| **Spy** | A real object, wrapped to record calls | When you want real behavior but also want to assert on calls |

**The rule of thumb:** prefer **fakes** for everything. Reach for `unittest.mock` only when you specifically need to assert "this was called with these args" *and* a fake can't tell you that. Mocks are over-used in Python codebases — they make tests brittle (they fail when implementation changes, even if behavior didn't).

The book's stance: build a `FakeRepository` once, use it in 200 tests. That's better than 200 tests each doing their own `mock.patch()`.

## Pytest anatomy — what you actually type

### AAA / Given-When-Then

Every test has three parts. Make them visible.

```python
def test_create_list_assigns_slug():
    # Arrange (Given)
    uow = FakeUnitOfWork()

    # Act (When)
    lst = services.create_list(user_id="u1", name="My Stuff", uow=uow)

    # Assert (Then)
    assert lst.slug == "my-stuff"
```

You don't need the comments — but the *shape* should be obvious. If you can't tell where Arrange ends and Act begins, the test is too tangled.

### Fixtures — shared setup

```python
@pytest.fixture
def uow():
    return FakeUnitOfWork()

def test_create_list_assigns_slug(uow):
    lst = services.create_list("u1", "My Stuff", uow)
    assert lst.slug == "my-stuff"
```

Pytest sees the `uow` argument and matches it to the fixture. Use this for anything you'd otherwise repeat 10 times across tests. Don't use it as a hiding place — fixtures that do too much become magic.

### Parametrize — same test, many cases

```python
@pytest.mark.parametrize("name,expected_slug", [
    ("My Stuff", "my-stuff"),
    ("Hello, World!", "hello-world"),
    ("   leading spaces", "leading-spaces"),
    ("UPPERCASE", "uppercase"),
])
def test_slug_from_name(name, expected_slug):
    assert List.slug_from_name(name) == expected_slug
```

Four tests for the price of one definition. Use this whenever you have multiple input cases for the same logic.

### Markers — categorize tests

```python
@pytest.mark.integration
def test_real_db_query(): ...

@pytest.mark.slow
def test_long_running(): ...
```

Then run only fast tests during dev: `pytest -m "not slow and not integration"`. Orqestrator does this — see its `pyproject.toml`.

## What makes a good test

Six properties. The first letters spell **F.I.R.S.T.** if you want a mnemonic, but I'll just list them.

1. **Fast** — milliseconds, not seconds. Slow tests don't get run.
2. **Isolated** — no shared state with other tests. Order shouldn't matter. Run alone, run in any order, same result.
3. **Repeatable** — deterministic. No flakes. If it depends on time/random/network, control those.
4. **Self-checking** — passes or fails on its own. No "look at the output and decide."
5. **Timely** — written close to the code, not three weeks later when the design has solidified.
6. **Readable** — the test name describes the behavior; the body shows arrange/act/assert clearly. Should read like a spec.

A good test name: `test_creating_list_with_empty_name_raises_invalid_list_name`.
A bad test name: `test_create_list_2`, `test_works`.

## The architecture connection (why this matters now)

The pyramid only works if your code lets it. Look what the layered architecture buys you:

| Layer | Test type | Why it's cheap |
|---|---|---|
| `domain/` | Unit (fast) | Pure Python, no I/O, no setup |
| `services/` | Service (fast) | Inject `FakeUnitOfWork` |
| `adapters/` | Integration (slower) | Real DB, but only for the adapter — not the whole app |
| `entrypoints/` | E2E (slowest) | TestClient + full stack — but only a few |

If your domain logic is mixed into routes (no service layer), every test is an E2E test. The pyramid inverts. Tests get slow. People stop running them. Coverage rots.

This is why **architecture and testing are the same conversation.** Bad architecture makes good tests impossible.

## Common traps

### Trap 1: testing implementation, not behavior

```python
# Bad — tests how it's done
def test_create_list_calls_repo_add():
    repo = Mock()
    services.create_list("u1", "x", repo)
    assert repo.add.called

# Good — tests the outcome
def test_create_list_persists_the_list():
    repo = InMemoryListRepository()
    services.create_list("u1", "x", repo)
    assert len(repo.list_for_user("u1")) == 1
```

The first test breaks if you rename `add` to `save`. The second only breaks if behavior actually breaks. **Test what the user observes, not how the code achieves it.**

### Trap 2: over-mocking

```python
# Bad — mocks everything
def test_send_message():
    repo = Mock()
    llm = Mock()
    user = Mock()
    notifier = Mock()
    services.send_message(...)
    # 14 mock assertions later...
```

When everything is a mock, the test isn't testing your code — it's testing your mocks. Use real domain objects + fake adapters. Mock only what you literally cannot construct.

### Trap 3: tests that depend on test order

```python
def test_a():
    db.users.insert(...)
def test_b():
    assert db.users.count() == 1   # ← only passes if test_a ran first
```

Each test should set up its own world and leave it clean. `FakeUnitOfWork` makes this trivial — fresh dict per test.

### Trap 4: testing every getter/setter

```python
def test_list_name_property():
    lst = List(name="x")
    assert lst.name == "x"
```

Pointless. Test **behavior**, not data plumbing. Tests that exist only to bump coverage are noise.

### Trap 5: test files that mirror class structure, not behavior

```
test_list.py
test_list_service.py
test_list_repository.py
test_list_dto.py
```

vs.

```
test_creating_a_list.py
test_sharing_a_list.py
test_listing_user_lists.py
```

The second organizes by **what the user does**, which makes failures readable: "creating a list" broke. Often more useful than "List class" broke.

(Both shapes are defensible. Just don't slavishly mirror the implementation if it makes failures harder to read.)

### Trap 6: flaky tests left alone

A flaky test (passes sometimes, fails sometimes) is **worse than no test.** It trains the team to ignore failures. Either fix it or delete it — never `@pytest.mark.skip` and forget. Triage every flake.

## Pragmatic principles — when to test what

The "schools of thought" disagreement is real. The honest senior practice:

**Always test:**
- Domain rules with non-trivial logic (`subscription.is_active()`, `list.add_item()` invariants)
- Services that touch money, auth, or data integrity
- Bug fixes — write a regression test before fixing
- Anything that would be expensive to debug from a prod log

**Often skip:**
- Throwaway prototypes / experiments
- Pure UI experiments
- Code you're definitely going to delete
- Trivial CRUD with no logic (the integration test for the framework is enough)

**The cost-vs-bug calculation:**
> *Cost of writing the test* vs. *cost of a bug × probability of a bug.*

Auth bug → catastrophic. Slug formatting bug → annoying. Test the auth path more.

**On TDD specifically:**
Strict TDD ("test first, watch it fail, make it pass, refactor") is one technique among many. Use it when:
- The behavior is well-specified up front (parser, calculator, validator)
- You're fixing a bug (write the regression first — much easier than after)
- You don't yet know the right API and want pressure to design one

Skip strict TDD when:
- You're exploring (you don't know what the right design is yet)
- The integration is the main risk (write the integration test first instead)

The pragmatic middle: **write tests close to the code, not strictly before.** What matters is *that they exist when you ship*, not *the order they were written in*.

## Self-check

Try answering in your own words:

1. Why is the test pyramid shaped that way? What goes wrong if it's inverted?
2. What's the difference between a fake and a mock? When do you reach for each?
3. Your service test needs a real Postgres to run. What's wrong, architecturally?
4. A test passes when run alone but fails in the full suite. What's the most likely cause?
5. Why is a flaky test worse than no test?
6. You're at 41% coverage on a real production codebase (orqestrator). Is that a problem? When?

## Things to steal

- **The pyramid is the architecture, viewed from the test side.** If the pyramid inverts, the architecture is broken.
- **Build one fake per port. Use it everywhere.** `FakeListRepository`, `FakeUnitOfWork`, `FakeNotifier`. Reuse beats reinvention.
- **Test behavior, not implementation.** If you rename a method, only the implementation tests should break.
- **Test names are specs.** `test_creating_list_with_empty_name_raises_invalid_list_name` reads like a requirement. `test_2` reads like nothing.
- **Coverage % is a vanity metric. Refactor confidence is the real metric.** Aim for "I can refactor this without fear," not "95%."
- **Ratchet coverage up over time, don't slam a high bar on day 1.** Orqestrator is doing this — start where you are, only allow it to go up.
- **Markers + `-m` let you run fast tests in dev, full suite in CI.** That's how you keep the loop fast without sacrificing rigor.
- **Flaky tests get fixed or deleted, never ignored.** A flake is a broken test that lies about being broken.
- **Bug fix? Write the failing test first. Always.** Cheapest way to ensure regression.
