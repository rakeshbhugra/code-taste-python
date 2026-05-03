
# Project Structure for Layered Architecture

← [Back to index](./README.md)

What the directory layout looks like once you apply the core five concepts. Tailored for a Python/FastAPI backend like hypedar or tytona.

## The canonical layout

```
your_app/
├── src/
│   └── your_app/
│       ├── domain/                  # Pure business — no external imports
│       │   ├── lists.py             # List entity + rules
│       │   ├── users.py             # User entity
│       │   ├── subscriptions.py
│       │   └── exceptions.py        # Domain-specific errors
│       │
│       ├── ports/                   # Abstract interfaces (ABCs)
│       │   ├── repositories.py      # ListRepository, UserRepository
│       │   ├── auth.py              # AuthProvider
│       │   ├── notifications.py     # Notifier
│       │   ├── summarizer.py        # Summarizer
│       │   └── unit_of_work.py      # UnitOfWork
│       │
│       ├── adapters/                # Concrete implementations of ports
│       │   ├── repositories/
│       │   │   ├── sqlalchemy_lists.py
│       │   │   ├── sqlalchemy_users.py
│       │   │   └── in_memory.py     # used by tests
│       │   ├── auth/
│       │   │   ├── clerk.py
│       │   │   └── fake.py
│       │   ├── summarizer/
│       │   │   ├── litellm.py
│       │   │   └── fake.py
│       │   ├── notifications/
│       │   │   ├── resend.py
│       │   │   └── fake.py
│       │   ├── orm.py               # SQLAlchemy table defs / mappers
│       │   └── unit_of_work.py      # SqlAlchemyUnitOfWork
│       │
│       ├── services/                # Application services (use cases)
│       │   ├── lists.py             # create_list, share_list, ...
│       │   ├── users.py
│       │   ├── billing.py
│       │   └── summarization.py
│       │
│       ├── entrypoints/             # The outside edges of the app
│       │   ├── api/
│       │   │   ├── main.py          # FastAPI() app
│       │   │   ├── deps.py          # Depends() for routes
│       │   │   ├── routes/
│       │   │   │   ├── lists.py
│       │   │   │   ├── users.py
│       │   │   │   └── billing.py
│       │   │   ├── schemas/         # Pydantic request/response DTOs
│       │   │   │   ├── lists.py
│       │   │   │   └── users.py
│       │   │   └── exception_handlers.py
│       │   └── cli.py               # optional: Typer/Click commands
│       │
│       ├── bootstrap.py             # Wires adapters → services at startup
│       └── config.py                # Settings from env vars
│
├── tests/
│   ├── conftest.py
│   ├── unit/                        # Domain tests — no I/O, milliseconds
│   ├── service/                     # Service tests with FakeUnitOfWork
│   ├── integration/                 # Adapter tests against real DB/APIs
│   └── e2e/                         # Full-stack HTTP tests
│
├── alembic/                         # Migrations
├── docs/
├── pyproject.toml
├── Dockerfile
└── README.md
```

## The dependency rule (the only thing you must get right)

Read directories top-to-bottom in the diagram above. **Each layer can only import from layers above it in this list:**

```
domain      ←  imports from: nothing (only stdlib + dataclasses)
ports       ←  imports from: domain
adapters    ←  imports from: domain, ports, external SDKs
services    ←  imports from: domain, ports
entrypoints ←  imports from: services, ports, schemas
bootstrap   ←  imports from: everything (it's the wiring)
```

If `domain/lists.py` ever imports from `adapters/` or `entrypoints/`, the architecture is broken. This is the **dependency rule**: dependencies always point *inward toward the domain*. Adding an `import` is the moment you violate it.

A 3-line CI check (`grep` for forbidden imports) is enough to enforce this.

## Why each directory exists

| Directory | One-sentence purpose | If you delete it... |
|---|---|---|
| `domain/` | Holds the business rules and entities | You have no business — just CRUD on dicts |
| `ports/` | Defines the interfaces between domain and the world | You can't fake anything for tests |
| `adapters/` | Concrete implementations that talk to external systems | The app can't actually do anything |
| `services/` | One use case per function — the orchestration layer | Routes get fat with business workflow |
| `entrypoints/` | HTTP, CLI, workers — the edges of the app | The app has no way to be invoked |
| `bootstrap.py` | Builds adapters once, injects them into services | You're constructing dependencies inside services (DI hell) |
| `config.py` | Reads env vars, holds settings | Magic strings everywhere |

## What's load-bearing vs. what's flexible

**Load-bearing (don't change these):**

- The dependency rule (inward only)
- `domain/` is pure — no SDK imports, no Pydantic, no FastAPI, nothing external
- `ports/` are ABCs (or Protocols), not concrete classes
- `services/` take ports as arguments, never import adapters directly
- `entrypoints/` are thin — parse input, call service, render output

**Flexible (pick what fits):**

- The exact directory names. Some teams call `services/` → `service_layer/`, `entrypoints/` → `interfaces/` or `api/`, `ports/` → `interfaces/`. The names matter less than the separation.
- Whether `ports/` is its own folder or lives inside `domain/`. Both are defensible. I prefer separate because it makes the abstract surface visible.
- Whether `adapters/` groups by external system (`adapters/auth/`, `adapters/storage/`) or by kind (`adapters/repositories/`, `adapters/messaging/`). Both work. Group however your codebase grows naturally.
- One file per entity (`domain/lists.py`, `domain/users.py`) vs. one big `domain/model.py`. Start with one file; split when files get big.
- Pydantic schemas in `entrypoints/api/schemas/` vs. alongside routes. Either is fine.

## What this looks like compared to hypedar today

Quick diff between recommended and current hypedar shape:

| What hypedar has | What it should have | Why |
|---|---|---|
| `src/api/routes/` | ✓ keep — rename to `src/entrypoints/api/routes/` if you want the canonical name | Same idea |
| `src/services/` | ✓ keep, but make services thinner | Move biz rules into `domain/` |
| `src/services/ai/base.py` (port) + `litellm_service.py` (adapter) | Split: `ports/summarizer.py` + `adapters/summarizer/litellm.py` | Ports and adapters are different things; keep them in different folders |
| `src/core/auth_providers/` | Same: split into `ports/auth.py` + `adapters/auth/clerk.py` | Same as above |
| `src/models/` (ORM) + `src/schemas/` (Pydantic) | ORM → `adapters/orm.py`. Pydantic → `entrypoints/api/schemas/`. **Domain entities go in `domain/`** as a third type | ORM model ≠ Pydantic schema ≠ domain entity. They serve different layers |
| (no domain folder) | `domain/` with rules-on-entities | Where rules live |
| (no ports folder) | `ports/` with ABCs | The seam shape |
| (no UoW) | `adapters/unit_of_work.py` + `ports/unit_of_work.py` | Transaction boundary |
| (no bootstrap) | `bootstrap.py` | Wires it all up at startup |

The biggest mental shift: **three kinds of "model" exist, and they're not the same.**

```
domain/lists.py        — the business entity (List with rules)
adapters/orm.py        — the SQLAlchemy table mapping (the row shape)
entrypoints/api/schemas/lists.py  — the Pydantic request/response (the wire shape)
```

Most apps conflate these into one. The book argues they should be separate because they serve different layers, change for different reasons, and have different fields. Pragmatically: **start by separating domain from the other two. ORM and Pydantic can stay merged early on if it's not painful yet.**

## The test pyramid that comes for free

Once the layout is right, your test directories mirror it naturally:

| Test type | Location | What it tests | Speed |
|---|---|---|---|
| **Unit** | `tests/unit/` | Pure domain — `List.add_item` rules | milliseconds |
| **Service** | `tests/service/` | Service functions with `FakeUnitOfWork` | tens of milliseconds |
| **Integration** | `tests/integration/` | Real `SqlAlchemyListRepository` against a real DB | hundreds of ms |
| **E2E** | `tests/e2e/` | Full HTTP via `TestClient` | seconds |

The pyramid: **lots of unit, fewer service, even fewer integration, very few e2e.** Most logic lives in domain → most tests are unit → tests stay fast.

If you find yourself only writing E2E tests, the layering is wrong: your business logic isn't actually in domain, so there's nothing to unit-test.

## A pragmatic warning

This is the *destination* shape. Don't try to land here on day 1 of a new project — you'll over-engineer.

Reasonable progression for a new app:

1. **Day 1 — flat:** `routes/`, `models/`, `services/`. Use SQLAlchemy directly. No ports, no UoW. Get something working.
2. **When biz logic shows up in services:** Add `domain/`. Move rules onto entities.
3. **When tests get painful:** Add `ports/repositories.py` + `adapters/repositories/`. You can now fake the DB.
4. **When you touch multiple repos in one operation:** Add Unit of Work.
5. **When you add a second external system (LLM, payments):** Generalize to `ports/` + `adapters/`. The first one was just a Repository; now you have the pattern.

For an existing app like hypedar (in step 2-3 territory): refactor one route at a time. Don't big-bang it. Pick `create_list`, get it into the destination shape, learn from the friction, then do the next one.

## Things to steal

- **Three folders are doing the most work: `domain/`, `services/`, `adapters/`.** Everything else is supporting cast. If you only get those three right, the architecture is mostly right.
- **The dependency rule is the architecture.** Folders are just naming. The rule is what makes the layering real.
- **Three kinds of "model" exist** — domain, ORM, schema. Conflating them is the most common subtle mistake.
- **Bootstrap is where the magic happens.** One file builds the world. Everything else just receives what it needs.
- **Don't land here on day 1.** The structure is a destination, not a template. Earn each layer when the previous one breaks down.
