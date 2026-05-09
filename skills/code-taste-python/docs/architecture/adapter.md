
# Adapter (and Port)

← [Back to index](./README.md)

## One-line definition

An adapter is **a swappable seam between your code and an external system** — and a port is the **interface that defines the seam.**

If you understood Repository, you've already used the pattern. Adapter is just the generalization: *not all external systems are databases.*

## Port and Adapter — what each word means

The vocabulary trips people up. Two words, one pattern:

| Term | What it is | Example |
|---|---|---|
| **Port** | The *interface* the domain depends on. An abstract base class or Protocol. Lives near the domain. | `ListRepository(ABC)`, `Notifier(ABC)`, `LLMService(ABC)` |
| **Adapter** | A *concrete implementation* of the port that talks to a specific external system. | `SqlAlchemyListRepository`, `SlackNotifier`, `LiteLLMService` |

Mental model: the **port is the shape of the hole**, the **adapter is the plug that fits it**. Your domain code only knows about the hole. Plugs are interchangeable.

You'll also hear this called *ports and adapters* or *hexagonal architecture* — same pattern, different vocabulary.

## The problem it solves

Your app needs to talk to external systems: databases, payment providers, email services, LLM APIs, auth providers, message queues. If you call those directly from biz logic:

1. **You can't test biz logic without the external system.** Want to test "subscription gets cancelled when payment fails"? Now you need a Stripe sandbox.
2. **Swapping providers is a rewrite.** Switch Auth0 to Clerk? Every file that imports `auth0_sdk` is suddenly broken.
3. **External SDK shapes leak into your code.** You start passing around `stripe.Customer` objects in your services, even though "customer" is a domain concept that should be yours, not Stripe's.

The adapter pattern fixes all three by putting **one seam** between your domain and each external system.

## Repository is just one specific adapter

This is the key realization: a **Repository is a port-and-adapter pattern applied to storage.**

```
ListRepository (ABC)            ← port, near the domain
    ↑
    ├── SqlAlchemyListRepository  ← adapter for Postgres
    ├── InMemoryListRepository    ← adapter for tests
    └── JsonListRepository        ← adapter for the mock data file
```

Same shape, applied to LLMs:

```
LLMService (ABC)                ← port
    ↑
    ├── LiteLLMService            ← adapter for the litellm SDK
    ├── OpenAIService             ← adapter for the openai SDK
    └── FakeLLMService            ← adapter for tests (returns canned responses)
```

Same shape, applied to auth:

```
AuthProviderAdapter (ABC)       ← port
    ↑
    ├── ClerkAuthAdapter          ← adapter for Clerk
    ├── Auth0AuthAdapter          ← adapter for Auth0
    └── FakeAuthAdapter           ← adapter for tests
```

Once you've seen one, you've seen them all. **Repository is the gateway drug.**

## What it looks like

The shape is always the same: define the port, write the real adapter, write a fake for tests, inject at the boundary.

```python
# domain/ports.py — the port lives near the domain
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class SummaryRequest:
    text: str
    max_words: int

@dataclass
class Summary:
    text: str
    tokens_used: int

class Summarizer(ABC):
    @abstractmethod
    def summarize(self, request: SummaryRequest) -> Summary: ...


# adapters/litellm_summarizer.py — the real adapter
class LiteLLMSummarizer(Summarizer):
    def __init__(self, model: str):
        self._model = model

    def summarize(self, request: SummaryRequest) -> Summary:
        resp = litellm.completion(
            model=self._model,
            messages=[{"role": "user", "content": f"Summarize in {request.max_words} words: {request.text}"}],
        )
        return Summary(
            text=resp.choices[0].message.content,
            tokens_used=resp.usage.total_tokens,
        )


# adapters/fake_summarizer.py — the test adapter
class FakeSummarizer(Summarizer):
    def __init__(self, canned_response: str = "fake summary"):
        self._canned = canned_response
        self.calls: list[SummaryRequest] = []  # for asserting what was called

    def summarize(self, request: SummaryRequest) -> Summary:
        self.calls.append(request)
        return Summary(text=self._canned, tokens_used=42)


# services/lists.py — the service depends only on the port
def summarize_list(list_id: str, repo: ListRepository, summarizer: Summarizer) -> Summary:
    lst = repo.get(list_id)
    request = SummaryRequest(text=lst.contents_as_text(), max_words=50)
    return summarizer.summarize(request)
```

Three things to notice:

1. **The port defines its own DTOs** (`SummaryRequest`, `Summary`). Not `litellm.Response`. The domain shouldn't know what shape `litellm` returns. The adapter translates.
2. **The port lives near the domain.** The adapter lives in an `adapters/` (or `infrastructure/`) directory. Dependencies point *inward toward the port*, never outward to the adapter.
3. **The service takes the port, not the adapter.** `summarizer: Summarizer`, not `summarizer: LiteLLMSummarizer`. The wiring (which concrete adapter to use) happens at the *edge* of the app — typically in `main.py` or a DI bootstrap.

## Real examples — mapped to your projects

| Project | External system | Port (interface) | Real adapter | Test adapter |
|---|---|---|---|---|
| Hypedar | Auth provider | `AuthProviderAdapter` | `ClerkAuthAdapter` (already exists ✓) | `FakeAuthAdapter` |
| Hypedar | LLM | `SummarizationService` | `LiteLLMSummarizationService` (exists ✓) | `FakeSummarizer` |
| Hypedar | Storage | `ListRepository` | `SqlAlchemyListRepository` (TODO) | `InMemoryListRepository` |
| Hypedar | Email (future) | `Notifier` | `ResendNotifier` | `FakeNotifier` |
| Tytona | LLM | `LLMService` | `LiteLLMService` | `FakeLLMService` |
| Tytona | Storage | `ConversationRepository` | `SqlAlchemyConversationRepository` | `InMemoryConversationRepository` |

Hypedar has already gotten this right twice (auth and AI). The pattern transfers cleanly.

## What's in / what's out

**In the port:**
- The interface methods the domain needs (`summarize`, `send_email`, `charge_card`)
- The DTOs the methods use as inputs and outputs (in *your* shape, not the SDK's)

**In the adapter:**
- The actual SDK calls (`litellm.completion`, `stripe.Charge.create`)
- Error translation: convert provider-specific exceptions to your domain exceptions
- Format translation: convert your DTOs to the SDK's expected input, and the SDK's response back to your DTOs

**Out of both:**
- Business rules (those live in the domain — adapters don't decide *whether* to send an email, just *how*)
- Use case orchestration (that's the service)

## The traps

### Trap 1: leaking the SDK's types through the port

```python
# Wrong — port returns the SDK's type
class Summarizer(ABC):
    @abstractmethod
    def summarize(self, text: str) -> litellm.Response: ...
```

The whole point of the port is to *hide* the SDK. If the port's signature mentions `litellm.Response`, the SDK has leaked through. Define your own DTO and have the adapter translate.

### Trap 2: the adapter that's actually a service

```python
class StripeAdapter:
    def charge_user_for_subscription(self, user_id: str, plan: str):
        user = self.repo.get(user_id)                    # ← biz logic
        if user.has_active_subscription:                 # ← biz rule
            return
        amount = self.pricing.calculate(plan, user.tier) # ← orchestration
        stripe.Charge.create(...)
```

That's not an adapter — that's a service that happens to call Stripe. The adapter should be **dumb**: take inputs, call the SDK, translate the result. Orchestration belongs in the service.

The clean version:

```python
# adapter — just talks to Stripe
class StripePaymentAdapter(PaymentAdapter):
    def charge(self, request: ChargeRequest) -> ChargeResult: ...

# service — does the orchestrating
def charge_user_for_subscription(user_id, plan, repo, payments):
    user = repo.get(user_id)
    if user.has_active_subscription:
        return
    request = ChargeRequest(amount=..., customer=user.stripe_id)
    payments.charge(request)
```

### Trap 3: too many ports too early

You don't need a port for every external call. Add one when:
- You can name a real second implementation (a fake for tests counts), AND
- The thing you're calling has its own quirky shape worth hiding (most SDKs do)

You probably *don't* need an adapter for `os.path.join` or `json.dumps`. Standard library + pure-Python utilities don't need wrapping.

### Trap 4: wrapping a wrapper

`litellm` already abstracts over OpenAI/Anthropic/etc. Wrapping `litellm` in a `Summarizer` port adds a *second* layer of abstraction over the same problem. Sometimes that's worth it (you control the DTOs, you can fake it for tests, you can swap the abstraction itself someday). Sometimes it's ceremony. Decide knowingly. The honest test: *can I name a real reason I'd swap litellm itself?* If "for tests" is the only answer, a simpler test fixture might do the job without the layer.

## How to spot it in your code

Hypedar's `src/core/auth_providers/` is a textbook adapter:

```
auth_providers/
├── base.py            # AuthProviderAdapter ABC + NormalizedUserData DTO
├── clerk.py           # ClerkAuthAdapter — the real one
└── factory.py         # picks the right adapter based on config
```

That's correct. Replicate this shape for the next external system you add.

The places hypedar *hasn't* used adapters yet:

- **Storage** — `_load_mock_data` / `_save_mock_data` are an inline JSON adapter pretending to be a service. Wrap them: `JsonListRepository` (adapter) implementing `ListRepository` (port). When you switch to Postgres, write `SqlAlchemyListRepository` and change one wiring line.
- **Email/notifications** — when you add them, design the port first (`Notifier.send(NotificationRequest)`), then the adapter (`ResendNotifier`).

## The hidden benefit

The big one is **swappability for tests**, which we already covered. But there's a subtler benefit:

**Adapters force you to design your own DTOs.**

When you write `class SummaryRequest`, you're being forced to ask: *"what does my domain actually need from a summarizer?"* Not "what does litellm accept" — what do *I* need. The answer is usually a smaller, sharper interface than the SDK's.

That clarity propagates upward. Your services use cleaner DTOs because you took the time to define them. And the day you swap providers, you discover that you only ever needed three of the SDK's twelve fields.

This is the deeper purpose of *"depend on abstractions you control, not on external libraries"* (the book's quote from Chapter 3). The adapter is the mechanism; designing-your-own-DTOs is the practice.

## Self-check

Try answering in your own words:

1. What's the difference between a *port* and an *adapter*?
2. Repository is a special case of what?
3. Your service is calling `stripe.Charge.create(...)` directly. Name three things wrong with that.
4. You're wrapping `litellm` (which already abstracts over OpenAI/Anthropic) in your own `Summarizer` port. When does this earn its rent, and when is it ceremony?
5. If `Summarizer.summarize` returns a `litellm.Response`, why is that a smell even though "it works"?

If those land, you've got Adapter. Up next: Unit of Work — the last of the core five.

## Things to steal

- **One seam per external system, not five.** Auth gets one adapter. Payments gets one adapter. Don't stack ports on ports.
- **The port lives near the domain; the adapter lives near the infrastructure.** Dependencies point inward, always.
- **Define your own DTOs at the port boundary.** Never expose SDK types upward. The translation lives in the adapter.
- **Adapters are dumb.** They translate, they call, they translate back. They don't decide.
- **Always write a fake adapter alongside the real one.** That's what makes the abstraction worth the cost.
- **The rent test is real here too.** "I might swap providers someday" is not a reason. "I have a fake for tests today" is.
