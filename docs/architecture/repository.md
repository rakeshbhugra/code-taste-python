
# Repository

← [Back to index](./README.md)

## One-line definition

A repository is **the one place that knows how to load and save your domain entities** — and the only thing in the codebase that talks to the database.

Think of it as a *collection-shaped interface over your storage*. Code that uses it shouldn't be able to tell whether the data lives in Postgres, in JSON files, or in memory.

## The problem it solves

Without a repository, DB code leaks everywhere. You end up with `session.query(...)` and `await db.execute(...)` scattered across services, route handlers, even utility functions. Three pains follow:

1. **Tests are painful.** To test `ListService.create_list`, you have to spin up a real DB or mock SQLAlchemy. Both suck.
2. **Swapping storage is impossible.** Want to move from JSON files to Postgres? You're rewriting the whole app.
3. **Domain gets polluted.** ORM imports creep into entity files. Now `List` knows about SQLAlchemy. Domain isn't pure anymore.

A repository fixes all three by being **the single seam** between domain and storage.

## What it looks like

```python
# repositories/lists.py
class ListRepository(ABC):
    @abstractmethod
    def get(self, list_id: str) -> List | None: ...

    @abstractmethod
    def get_all_for_user(self, user_id: str) -> list[List]: ...

    @abstractmethod
    def add(self, lst: List) -> None: ...

    @abstractmethod
    def save(self, lst: List) -> None: ...


class SqlAlchemyListRepository(ListRepository):
    def __init__(self, session): self._session = session

    def get(self, list_id):
        return self._session.query(ListModel).filter_by(id=list_id).one_or_none()
    # ...

class InMemoryListRepository(ListRepository):
    def __init__(self): self._lists: dict[str, List] = {}

    def get(self, list_id):
        return self._lists.get(list_id)
    # ...
```

The interface is shaped like **a Python collection** — `get`, `add`, `save`, `list`. Not like a SQL query layer. That's deliberate: the caller thinks "I'm working with a collection of Lists," not "I'm writing a query."

## Real examples

| Project | What's the repository swap-case? | Justified? |
|---|---|---|
| Hypedar | JSON file → Postgres (you're literally going to do this soon) | ✓ Yes — the swap is real |
| Tytona | Postgres → ??? | Maybe not — if you'll always use Postgres, the abstraction earns less |
| `requests` | (no repository — no durable state to store) | N/A |
| Cosmic Python book | Postgres → in-memory dict for tests | ✓ Yes — fast tests are the killer feature |

The honest test: **can you name a real second implementation you'd write?** If yes, the repository pays rent. If you're inventing one ("maybe someday Mongo"), you're speculating.

For most apps, the second impl is `InMemoryListRepository` for tests. That alone usually justifies the layer.

## What's in / what's out

**In the repository:**
- Loading entities by id, by query (`get_active_for_user`, `find_by_email`)
- Saving / updating entities
- Deleting entities
- Translating between domain objects and ORM rows (if those are different — see "Trap 2" below)

**Out of the repository:**
- Business rules (those live on the entity)
- Transaction management (that's the Unit of Work's job — coming later)
- HTTP concerns (status codes, request parsing)
- Caching, retries, metrics (these go in decorators *around* the repository, not inside)

## The traps

### Trap 1: making the repository a thin SQL wrapper

Bad:
```python
class ListRepository:
    def execute_query(self, sql: str): ...
    def execute_filter(self, **filters): ...
```

That's not a repository — that's a database client with a fancy name. The whole point is to express **domain operations**, not generic queries. If your repo methods look like SQL, you've leaked the DB upward.

Good repo method names: `get_active_subscriptions_for_user`, `find_lists_by_slug`. Those are domain questions. The SQL is hidden inside.

### Trap 2: returning ORM objects instead of domain entities

If `repo.get(id)` returns a SQLAlchemy `ListModel` (with `.query`, `.session`, lazy-loaded relationships, etc.), you haven't decoupled anything. The service still has ORM tendrils everywhere.

Two options:
- **Easy:** use the ORM model *as* the domain entity. Pragmatic, common, fine for most apps. Your tradeoff: domain isn't 100% pure.
- **Strict:** keep `ListModel` (ORM) and `List` (domain) separate. Repository translates between them. Pure but more code.

The book recommends strict. Most real Python codebases do easy. Pick honestly based on whether your domain has rules complex enough to justify the translation cost.

### Trap 3: one repository per table

A repository is **per aggregate**, not per table. If `List` has many `ContentItem`s and items only make sense as part of a list, you have *one* `ListRepository` that loads the whole list-with-items together. You don't make a separate `ContentItemRepository`.

(Aggregate = a cluster of objects treated as one unit. We'll get to it later. For now: one repo per "root" entity.)

### Trap 4: adding a repository when you don't have a domain

If your "entity" is just a Pydantic schema with no behavior, your "repository" is just a CRUD wrapper, and your "service" just forwards calls to the repo — you've added three layers to do what one function could.

The cure isn't "don't use repositories." It's "wait until your domain has actual behavior worth protecting." For pure CRUD apps, the book itself says: just use Django.

## How to spot it in your code

In hypedar's `ListService`, look at `_load_mock_data` and `_save_mock_data`. Those are the repository — they just don't have a name yet. They sit *inside* the service instead of beside it.

Refactor sketch:

```python
# Before (today): service knows about JSON files
class ListService:
    @staticmethod
    def _load_mock_data():
        with open(MOCK_DATA_FILE) as f: return json.load(f)

    @staticmethod
    def get_all_lists(user_id):
        data = ListService._load_mock_data()
        return [...]

# After: service knows about a repository
class ListService:
    def __init__(self, repo: ListRepository):
        self._repo = repo

    def get_all_lists(self, user_id):
        return self._repo.get_all_for_user(user_id)

# Two impls, picked at wiring time:
class JsonListRepository(ListRepository): ...     # today
class SqlAlchemyListRepository(ListRepository): ... # tomorrow
```

The service code **doesn't change** when you swap JSON for Postgres. That's the win.

## The hidden benefit nobody mentions up front

You can write **fast tests** for the service layer using `InMemoryListRepository`. No DB, no fixtures, no migrations — just instantiate the repo with a dict and run. This single benefit is why most teams who adopt the pattern keep it.

```python
def test_create_list_assigns_slug():
    repo = InMemoryListRepository()
    service = ListService(repo)
    service.create_list(user_id="u1", name="My Stuff")
    assert repo.get_all_for_user("u1")[0].slug == "my-stuff"
```

That test runs in milliseconds. No `pytest-postgres`, no `docker-compose up`. That speed compounds when you have hundreds of tests.

## Self-check

Try to answer in your own words:

1. What's the difference between a repository and "the database"?
2. In hypedar, what's the *real* swap-case that makes a `ListRepository` worth adding?
3. Why is `repo.execute_sql("SELECT ...")` a smell? What's a method name that wouldn't be?
4. If your domain entity is just a Pydantic schema with no methods, does adding a repository help? Why or why not?

If you can answer all four, you've got Repository. Move on to Service.

## Things to steal

- **Express domain operations, not queries.** `get_active_subscriptions_for_user` beats `query(filters={"status": "active"})`. The method name tells the reader the *intent*.
- **One seam, not five.** All DB access goes through the repository. If a route handler imports SQLAlchemy directly, the seam has leaked.
- **InMemory implementations are the killer feature.** Even if you only ever use Postgres in production, having an InMemory impl for tests pays for the abstraction by itself.
- **Repository per aggregate, not per table.** Group by domain shape, not by schema shape.
- **Wait for the swap to be real before adding the layer.** "Mongo someday" is not a swap. "JSON now, Postgres in two months" is.
