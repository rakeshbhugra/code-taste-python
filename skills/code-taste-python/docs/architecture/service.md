
# Service (Service Layer)

← [Back to index](./README.md)

## One-line definition

A service is **the function that runs one use case from start to finish** — load the entities it needs, ask the domain to do its thing, persist the result.

It's the layer that says *"this is what happens when a user does X."*

## The problem it solves

Without a service layer, you have two bad places to put use-case logic:

1. **In the route handler (Flask/FastAPI).** Then your "use case" is mixed with HTTP parsing, status codes, and authentication. You can't reuse it from a CLI, a background worker, or a test without dragging Flask along.
2. **On the domain entity.** Then your entity ends up loading other entities, talking to repos, and orchestrating workflows — which is exactly the persistence-ignorance violation we just fixed.

The service layer is the home for **application-level orchestration** that's neither HTTP nor pure business rule. It's the answer to *"where does the workflow live?"*

## What it looks like

The classic shape — three steps, every time:

```python
def add_item_to_list(list_id: str, item_data: dict, repo: ListRepository) -> None:
    lst = repo.get(list_id)              # 1. load
    if lst is None:
        raise ListNotFound(list_id)
    lst.add_item(Item(**item_data))      # 2. delegate to domain
    repo.add(lst)                        # 3. persist (commit lives outside, see UoW later)
```

Notice what's *not* there:
- No `if len(lst.items) >= MAX:` — that's a domain rule, lives on `List`
- No `@router.post(...)` — that's HTTP, lives in the route
- No `session.commit()` — that's transaction management, lives in UoW (next concept)

The service is the **glue** between layers. It coordinates; it doesn't decide.

## "Service" vs "Domain Service" — the confusing bit

The word *service* gets used for two different things:

| Type | Lives in | Example | Job |
|---|---|---|---|
| **Application service** (what this doc means) | `services/` | `add_item_to_list(...)` | Orchestrates a use case across domain + repo |
| **Domain service** | `domain/` | `allocate(line, batches)` | A piece of business logic that doesn't naturally belong on a single entity |

The book itself flags this confusion (Chapter 4 has a section literally called *"Why Is Everything Called a Service?"*).

When this doc says "service," it means **application service**. Domain services are rarer — you reach for one only when a rule involves multiple entities and there's no obvious one to put it on.

## Real examples — mapped to your projects

| Use case | Service method | What it orchestrates |
|---|---|---|
| Hypedar: user creates a list | `create_list(user_id, name, repo)` | Generate slug (domain), make `List` entity, persist via repo |
| Hypedar: user adds item to list | `add_item_to_list(list_id, item, repo)` | Load list, call `lst.add_item()`, save |
| Tytona: user sends a message | `send_message(conversation_id, content, llm, repo)` | Load conversation, append turn, call LLM adapter, persist |
| Hypedar: summarize a list | `summarize_list(list_id, summarizer, repo)` | Load list, call summarizer adapter, save summary |

Notice each one is **one verb from the user's perspective**. That's the test for "is this a service method?" — it should map to a thing the user (or another system) wants to do.

## What's in / what's out

**In a service:**
- Loading entities (`repo.get`)
- Calling domain methods (`lst.add_item(...)`)
- Persisting (`repo.add`)
- Orchestrating across multiple aggregates (load List, load User, do thing, save both)
- Calling adapters for external systems (LLM, payment, email)
- Authorization checks (debatable — some teams put on domain)
- Translating between input dicts/DTOs and domain objects

**Out of a service:**
- Business rules (those live on the entity — `lst.add_item` decides if the item is allowed, not the service)
- HTTP concerns (status codes, request parsing, CORS — those live in the route)
- DB queries (those live in the repository)
- Transaction management (next concept: Unit of Work)

## The traps

### Trap 1: the "fat service" with logic that should be on the domain

```python
# Wrong — biz rule in service
def add_item_to_list(list_id, item, repo):
    lst = repo.get(list_id)
    if len(lst.items) >= 100:                # ← business rule!
        raise ListFull()
    if any(i.id == item.id for i in lst.items):  # ← business rule!
        raise DuplicateItem()
    lst.items.append(item)
    repo.add(lst)

# Right — service orchestrates, domain decides
def add_item_to_list(list_id, item, repo):
    lst = repo.get(list_id)
    lst.add_item(item)    # ← rules enforced inside add_item
    repo.add(lst)
```

The smell: services with lots of `if`s about *the business*. Move them onto the entity.

### Trap 2: the "anemic service" that just forwards to the repo

```python
def get_list(list_id, repo):
    return repo.get(list_id)
```

If your service does literally one repo call and nothing else, the service layer isn't earning its rent for that operation. Either:
- Drop it and let the route call the repo directly (pragmatic), or
- Accept the slight ceremony for consistency (defensible — every operation goes through services, no exceptions)

The book recommends the latter — *"clearly define where our use cases begin and end"* is a stronger property than "no useless one-liners." But know which choice you're making.

### Trap 3: services calling services

```python
def add_item_to_list(list_id, item, repo):
    list_service.validate_list_exists(list_id)   # ← service calling service
    item_service.normalize_item(item)            # ← service calling service
    ...
```

Once you have services calling services, you've got a coordination layer above your coordination layer. Pull the helpers down to either domain (if it's a rule) or to a private function in the same module (if it's a workflow step). Services should be **flat**.

### Trap 4: putting routes' job in the service

```python
def create_list(request: Request):                 # ← service taking a Request?
    user_id = request.state.user.id
    data = request.json()
    ...
```

A service shouldn't know what an HTTP request looks like. Take primitives or DTOs (`user_id: str`, `data: CreateListRequest` Pydantic), not framework objects. That's how you get to call the same service from a CLI or a worker.

## How to spot it in your code

Look at hypedar's `src/api/routes/lists.py` and `src/services/lists.py`:

- **Routes are correctly thin** ✓ — they parse input, call a service, return JSON. Good.
- **Service is doing too much** — `ListService.create_list` does slug generation, dict construction, file I/O, and persistence. Slug generation is a domain rule (move to `List.slug_from_name()`). File I/O is repository work (move to `JsonListRepository`). What's left in the service is the right shape: load → mutate → save.

Sketch of the cleaned-up version:

```python
# domain/list.py
@dataclass
class List:
    id: str
    name: str
    slug: str
    items: list[Item]

    @staticmethod
    def slug_from_name(name: str) -> str:
        return re.sub(r'[^a-z0-9]+', '-', name.lower()).strip('-')

    def add_item(self, item: Item) -> None:
        if len(self.items) >= 100:
            raise ListFull()
        self.items.append(item)


# services/lists.py
def create_list(user_id: str, name: str, repo: ListRepository) -> List:
    lst = List(
        id=str(uuid4()),
        name=name,
        slug=List.slug_from_name(name),
        items=[],
    )
    repo.add(user_id, lst)
    return lst
```

The service became a 4-line function. The rules went home (to `List`). The I/O went home (to `repo`). That's the shape to aim for.

## The hidden benefit

Services define your **public API in business terms**, independent of transport.

Once you have them, this just works:

```python
# From a Flask route
@router.post("/lists")
def create_list_endpoint(req: CreateListRequest, repo=Depends(get_repo)):
    return services.create_list(req.user_id, req.name, repo)

# From a CLI
@cli.command()
def make_list(user_id, name):
    services.create_list(user_id, name, repo=get_repo())

# From a background worker
def on_user_signup(event):
    services.create_list(event.user_id, "Welcome list", repo=get_repo())

# From a test
def test_creating_a_list_assigns_slug():
    repo = InMemoryListRepository()
    lst = services.create_list("u1", "My Stuff", repo)
    assert lst.slug == "my-stuff"
```

Same use case, four entry points, zero duplication. That's the payoff.

## Self-check

Try answering in your own words:

1. What's the difference between a *route handler* and a *service*?
2. What's the difference between an *application service* and a *domain service*?
3. If a service method is just `return repo.get(id)`, is something wrong? Why or why not?
4. In the three-step shape (load → delegate → save), why does the *delegate* step happen on the domain entity rather than in the service?
5. Could you call your service from a Celery task? If not, what's leaking?

If those land, you've got Service. Up next: Adapter.

## Things to steal

- **Three steps, in order: load, delegate, save.** If your service method has a different shape, ask why.
- **One service function = one use case.** Named for what the user does, not for what the data looks like. `create_list`, not `insert_into_lists_table`.
- **Take primitives or DTOs, not framework objects.** That's the difference between "service" and "controller with extra steps."
- **Services should be flat.** Service-calling-service is a smell — pull the helper down.
- **The biz rule test:** if there's an `if` about business logic in your service, it almost certainly belongs on the entity.
- **The transport test:** could you call this service from a CLI tomorrow? If not, something HTTP-shaped has leaked in.
