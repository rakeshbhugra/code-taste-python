# Domain

← [Back to index](./README.md)

## One-line definition

The domain is **what your app is fundamentally about, in business terms — not the tech.**

If a non-developer asked you "what does your app do?", the words you'd use are domain words.

## What's the domain in real projects?

| Project | Domain | Domain concepts | Example domain rules |
|---|---|---|---|
| Tytona | Conversations with LLMs | `Conversation`, `Message`, `Turn`, `User`, `LLMModel` | "a turn has one user message + one assistant message"; "a conversation belongs to one user" |
| Hypedar | Content lists and subscriptions | `List`, `ContentItem`, `Source`, `Subscription`, `User` | "a free user can have at most N lists"; "a subscription is active if not expired AND paid" |
| `requests` | HTTP | `Request`, `Response`, `Session` | "an HTTP method is one of GET/POST/PUT/..."; "a redirect chain has a max length" |

Notice: **none of these mention Postgres, FastAPI, SQLAlchemy, or LiteLLM.** Those are all *infrastructure*. The domain is what would still exist if you ported the app to a different language or framework.

## What lives in the domain *layer* (the actual code)

- **Entities** — the "things." `Conversation`, `List`, `Subscription`. Usually a class.
- **Domain logic** — the *rules* about those things, written as methods on the entities themselves.

Example: a `Subscription` entity in hypedar today is probably just an SQLAlchemy class with fields. A rich-domain version would have a method that captures a business truth:

```python
class Subscription:
    def is_active(self) -> bool:
        return self.status == "paid" and self.expires_at > datetime.now(UTC)
```

That `is_active` rule is *domain logic*. Not "how to query the DB" or "how to render JSON" — a *business truth about a subscription*. So it belongs on the entity.

## What does NOT belong in the domain

- DB queries (`session.query(...)`)
- HTTP/FastAPI stuff (`@router.get`, `Depends`)
- External API calls (`litellm.completion`, `stripe.charge`)
- File paths, env vars, config

If you imported any of those into a domain class, you've polluted the domain.

## The trap: anemic vs rich domain models

There are two ways to organize biz logic, and this is the single most important distinction in this whole topic.

**Anemic domain model** (what most code in the wild looks like):
```python
class List:                        # just data
    id: str
    name: str
    items: list

class ListService:                 # all logic lives here
    def add_item(list, item): ...
    def can_add_more(list): ...
```

**Rich domain model** (what the book recommends):
```python
class List:                        # data + rules
    def add_item(self, item): ...
    def can_add_more(self): ...

class ListService:                 # just orchestrates
    def add_item_to_list(self, list_id, item):
        lst = repo.get(list_id)
        lst.add_item(item)         # ← domain logic on the entity
        repo.save(lst)
```

In hypedar today, your code is **anemic**: `ListService` has the logic, `List` (the model) is just fields. That's not wrong — most apps are like this — but the book argues anemic models are a smell when the domain has real rules.

## How to spot it in your code

Open one of your services and ask: "is anything in this method *only* about business rules, with no DB/HTTP stuff?" If yes, **that piece probably belongs on the entity.**

Concrete example from hypedar's `ListService.create_list`:

```python
slug = ListService._generate_slug(list_data.name)
```

A slug is a *business rule about how lists name themselves*. There's no DB or HTTP here. So `_generate_slug` arguably belongs as a class method on `List` itself, not as a private static on the service. The service should just orchestrate, not contain rules.

## Self-check

Try to answer in your own words:

1. What's "the domain" of tytona?
2. Why is `litellm.completion(...)` *not* domain code?
3. What's the difference between anemic and rich domain models?

If you can answer all three without looking, you've got Domain. Move on to Repository.

## Things to steal

- The mental separation of "what my app is fundamentally about" vs "the tech that happens to power it." Even if you never write a single domain class, holding this distinction in your head will make every other architecture decision easier.
- The anemic-vs-rich distinction. Even when you choose anemic (which is fine for CRUD), do it knowingly — not because you didn't know the alternative existed.
