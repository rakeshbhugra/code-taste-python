
# Logging & Observability

← [Back to index](./README.md)

Logging is one of those things that's easy to do badly for years before you realize. This doc is the orientation: the recommended stack for Python/FastAPI in 2026, the few concepts that matter, and the traps.

## The goal (the only one)

> **When something goes wrong in prod, you should be able to answer "what happened to *this* user's request?" in under 60 seconds.**

That's it. Coverage of every line of code, fancy formatters, log volume — none of it matters if you can't answer that question. Conversely, a small amount of *structured* logging plus correlation IDs lets you answer it cheaply.

## The recommended stack

Three layers, each doing one job:

| Layer | Tool | Job |
|---|---|---|
| **Tracing / spans / auto-instrumentation** | [Logfire](https://logfire.pydantic.dev/) (or OpenTelemetry directly) | Per-request traces, span breakdowns, auto-instrumentation of FastAPI/SQLAlchemy/httpx |
| **Structured app logs** | [`structlog`](https://www.structlog.org/) | Business events, debug context, audit logs — anything that isn't a span |
| **Correlation ID middleware** | [`asgi-correlation-id`](https://github.com/snok/asgi-correlation-id) | Per-request ID that follows the request through every log line and span |

Logfire is built on OpenTelemetry — they're not competitors. Logfire is the opinionated Python wrapper. If you're a Python/FastAPI shop, default to Logfire. If you're polyglot or already have an OTel collector, use OTel directly.

**You already have Logfire in orqestrator.** That covers tracing. The piece most teams miss is *adding `structlog` + correlation-ID middleware on top* — and that's where 80% of the practical "I can debug a prod issue" value lives.

## Key concepts

### 1. Structured logging — JSON / key-value, not f-strings

This is the single biggest upgrade. The difference:

```python
# Bad — grep-able only
logger.info(f"User {user.id} created list {list.id} with slug {slug}")

# Good — queryable
logger.info("list_created", user_id=user.id, list_id=list.id, slug=slug)
```

The good version lets you query *"all `list_created` events for `user_id=42` in the last hour"* in Logfire/Datadog. The bad version requires grepping a log file.

Rules of thumb:
- **Event names** are `snake_case` verbs in past tense: `list_created`, `subscription_upgraded`, `payment_failed`
- **Fields** are key-value, never embedded in strings
- **Same event name = same shape.** Don't have `list_created` with `user_id` in some places and `user` in others

### 2. Correlation IDs propagate across logs and spans

Every request gets a unique ID at the edge (or honors an incoming `X-Request-ID` header from upstream). That ID is bound to the request's logging context AND attached as a span attribute. Then *every* log line for that request includes it automatically — you don't pass it through every function.

The standard library for this is `asgi-correlation-id` — set-and-forget, works with structlog out of the box. With it set up, your logs look like:

```
{"event": "list_created", "request_id": "01h8x...", "user_id": 42, ...}
{"event": "summarizer_called", "request_id": "01h8x...", "tokens": 1200, ...}
{"event": "request_completed", "request_id": "01h8x...", "duration_ms": 820, ...}
```

You can pivot from one log line to all log lines for the same request in a single query.

### 3. Use ASGI middleware, NOT `@app.middleware("http")`

This is a real footgun. FastAPI's `@app.middleware("http")` decorator runs in a different async context than your endpoints — `contextvars` set in the middleware are *lost* by the time your endpoint runs.

```python
# Bad — context vars don't propagate to endpoints
@app.middleware("http")
async def add_request_id(request, call_next):
    request_id_ctx.set(generate_id())   # ← lost in endpoint
    return await call_next(request)

# Good — pure ASGI middleware
app.add_middleware(CorrelationIdMiddleware)   # from asgi-correlation-id
```

Use pure ASGI middleware (the `app.add_middleware(...)` form) for anything that needs to share context with the route. `asgi-correlation-id` already does this correctly — that's why we use it.

### 4. structlog routes *through* stdlib, not around it

You don't pick "structlog OR stdlib" — you wire structlog **as the formatter for stdlib**. That way Uvicorn's own logs and your app's logs both come out structured, and you keep all the stdlib handler ecosystem (Datadog handler, CloudWatch handler, file rotation, etc.).

The shape:

```
your code calls       structlog.get_logger().info(...)
                                  ↓
                         structlog processors
                                  ↓
                         stdlib logger / handlers
                                  ↓
                         JSON to stdout / Datadog / CloudWatch
```

Uvicorn and other libraries call `logging.getLogger()` directly — those go through stdlib too, get the same JSON formatting, and end up in the same place.

### 5. Background tasks need explicit context propagation

When you `BackgroundTasks.add_task(...)` in FastAPI, the task runs *after* the response has gone out. By then, the middleware has torn down its context — meaning the task starts with empty context (no request_id, no user_id).

The fix: capture context while inside the request, pass it to the task explicitly, re-bind in the task wrapper.

```python
def background_send_email(request_id: str, user_id: int):
    structlog.contextvars.bind_contextvars(request_id=request_id, user_id=user_id)
    logger.info("email_send_started")
    # ...

@router.post("/lists")
async def create_list(req, bg: BackgroundTasks):
    request_id = correlation_id.get()    # capture while still in request
    user_id = current_user.id
    bg.add_task(background_send_email, request_id, user_id)
```

## Logs vs. traces — the distinction

Often confused. They answer different questions.

| | Logs | Traces |
|---|---|---|
| **Question answered** | "what happened?" | "where did the time go?" |
| **Shape** | Discrete events with fields | Time-ordered spans nested by call hierarchy |
| **Always emitted?** | Yes | Sometimes sampled |
| **Best for** | Business events, debug, audit | Latency analysis, request flow visualization |
| **Tool** | structlog → JSON → log aggregator | Logfire / OpenTelemetry |

You want both. Logfire actually unifies them — your `logger.info(...)` calls and your spans appear in the same UI, correlated by trace ID. That's the killer feature.

## What to log vs. not log

**Always log:**
- Request boundary: method, path, status, duration, user_id, request_id
- Business events: `list_created`, `subscription_upgraded`, `payment_charged`
- Errors: with full traceback + structured context (which user, which entity, which input)
- External call durations: LLM, payment provider, third-party APIs — anything you don't control

**Never log:**
- Passwords, API keys, JWT tokens (even "just for debug")
- Full request/response bodies if they contain PII
- Credit card numbers, even partial
- Personal data without thinking about GDPR / your jurisdiction's privacy laws

**Be careful with:**
- Email addresses — sometimes PII, depends on jurisdiction
- Free-text user input — could contain anything; sanitize or skip
- IP addresses — counts as PII in EU

The pragmatic rule: **if it would embarrass you in a data breach, don't log it.**

## What it looks like — minimum viable setup

This is the smallest config that gives you all the wins. Drop into a new FastAPI app and you're done.

```python
# logging_config.py
import logging
import structlog
from asgi_correlation_id.context import correlation_id

def configure_logging(env: str = "dev"):
    timestamper = structlog.processors.TimeStamper(fmt="iso")

    shared_processors = [
        structlog.contextvars.merge_contextvars,        # request-scoped context
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        timestamper,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        # Inject correlation ID into every log line
        lambda _, __, event_dict: {
            **event_dict,
            "request_id": correlation_id.get(),
        },
    ]

    structlog.configure(
        processors=[
            *shared_processors,
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        foreign_pre_chain=shared_processors,
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            structlog.processors.JSONRenderer() if env != "dev" else structlog.dev.ConsoleRenderer(),
        ],
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)
    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(logging.INFO)
```

```python
# main.py
from fastapi import FastAPI
from asgi_correlation_id import CorrelationIdMiddleware
from logging_config import configure_logging

configure_logging(env="dev")
app = FastAPI()
app.add_middleware(CorrelationIdMiddleware)
```

```python
# In a service
import structlog
logger = structlog.get_logger()

def create_list(user_id: str, name: str, uow):
    with uow:
        lst = List.create(name=name)
        uow.lists.add(lst)
        uow.commit()
    logger.info("list_created", user_id=user_id, list_id=lst.id, slug=lst.slug)
    return lst
```

That's the whole thing. Every log line will now have `request_id`, `timestamp`, `level`, plus whatever fields you pass. JSON in prod, pretty-printed in dev.

## The traps

### Trap 1: f-string logs that pretend to be structured

```python
logger.info(f"created list {list_id} for user {user_id}")
```

This is grep-only. The fix is mechanical but tedious — replace with `logger.info("list_created", list_id=..., user_id=...)`. Do it once and never write f-string logs again.

### Trap 2: same event name with different shapes

```python
# in services/lists.py
logger.info("list_created", user_id=42, list_id=...)

# in services/admin.py
logger.info("list_created", admin=..., target_user=42, list=...)
```

Now your dashboard breaks because `user_id` exists sometimes and `target_user` other times. **Same event name = same shape.** If shapes differ, the event is different — name them differently.

### Trap 3: logging at the wrong level

```python
logger.error("user_not_found", user_id=user_id)   # ← is this an error?
```

If the user just typo'd a URL, that's not an error — it's an expected 404. Errors should mean "something is broken or needs attention."

Rough guide:
- `DEBUG` — only useful when debugging a specific issue. Off in prod by default.
- `INFO` — normal operations. Business events, request boundaries.
- `WARNING` — unexpected but recoverable. Rate-limited fallback used, retry succeeded after failure.
- `ERROR` — something is broken. Page-the-on-call territory if it's frequent.
- `CRITICAL` — the app is degraded or down.

### Trap 4: logging too much

Every line of log costs money (storage, indexing, network). Logging on every list-item lookup at INFO is fine for 100 RPS, ruinous at 100k. Sample debug logs in hot paths. Log at the *boundary* of work, not inside the loop.

### Trap 5: leaking secrets in error logs

```python
except Exception as e:
    logger.error("api_call_failed", request=request_data)   # ← request_data might have the API key
```

Secrets often live in request bodies, headers, or config. Sanitize before logging. structlog has processors for this; bigger codebases write a `redact_sensitive` processor that runs first.

### Trap 6: using `print()`

```python
print(f"DEBUG: list created {list_id}")
```

`print` doesn't honor log levels, doesn't get formatted as JSON, doesn't get a `request_id`, doesn't go to your aggregator, can't be filtered. **Never use print in production code.** Even in tytona's old code (`utils/postgres_crud_helper`), the `print(f"LiteLLM Error...")` pattern is one of the loudest "junior code" smells. Use the logger.

### Trap 7: not logging request boundaries

You should always be able to find *the start and end of every request* in logs. If you only log inside service methods, a request that errors before reaching a service is invisible. The middleware is what guarantees this — log `request_started` and `request_completed` at the boundary.

## How to spot it in your code

Quick smell test for your existing code:

| Symptom | Fix |
|---|---|
| `print(...)` calls anywhere | Replace with `logger.info(...)` |
| `logger.info(f"...")` with embedded values | Replace with structured fields |
| Same event name with different shapes across files | Pick one shape, refactor |
| Errors logged without traceback | Add `exc_info=True` or use structlog's `format_exc_info` processor |
| You can't search for "all events from user_id=42" | Missing structured fields or correlation ID |
| Background tasks have no request_id | Need explicit context propagation |
| Logs in dev are JSON (hard to read) | Use `ConsoleRenderer` in dev, `JSONRenderer` in prod |

## Pragmatic upgrade path

You don't need everything at once. Add in this order:

| Stage | What to add | Effort |
|---|---|---|
| 1 | `asgi-correlation-id` middleware. Now every request has an ID. | 5 min |
| 2 | structlog with stdlib integration + JSON output in prod. | 30 min |
| 3 | Replace all `print()` and `f-string` logs with structured calls. | 1-2 hours per project |
| 4 | A request-boundary middleware (logs `request_started` / `request_completed` with duration). | 30 min |
| 5 | Logfire (or OTel) for tracing. Auto-instrument FastAPI + SQLAlchemy. | 30 min — already done in orqestrator |
| 6 | Sensitive-field redaction processor. | 30 min |
| 7 | Per-environment log level + sampling for hot paths. | 1 hour |

For **qhive** right now (it's tiny): stages 1, 2, 4 are enough. ~1 hour total. Don't go heavy on a 1-endpoint app.
For **hypedar** if you're refactoring: stages 1-4 are the high-leverage ones.

## Self-check

Try answering in your own words:

1. Why is `logger.info(f"User {id} did X")` worse than `logger.info("user_did_x", user_id=id)`?
2. What does a correlation ID give you that timestamps alone don't?
3. Why is `@app.middleware("http")` a footgun for setting context vars?
4. structlog or stdlib `logging` — which do you choose? (Trick question.)
5. Logs and traces — what's the difference, and why do you want both?
6. You see "user_not_found" at level ERROR. Right or wrong, and why?

## Things to steal

- **Structured > grep-able.** JSON / key-value in prod, pretty in dev. Once you've worked in a codebase with structured logs, going back is painful.
- **Correlation ID at the edge, propagated everywhere.** One ID, many log lines, single query to find them all.
- **Same event name = same shape.** Treat event names like API contracts.
- **structlog routes *through* stdlib.** You get structured logs *and* the handler ecosystem.
- **Pure ASGI middleware for context vars.** `@app.middleware("http")` is broken for this.
- **Background tasks need explicit context.** The middleware is gone by then.
- **Logs answer "what happened?" — traces answer "where did time go?"** You want both.
- **Never `print()` in production code.** Loudest junior smell.
- **If it would embarrass you in a data breach, don't log it.** Pragmatic PII rule.
- **The pyramid of observability is the architecture, viewed sideways.** Clean spans + structured logs + correlation IDs only work if your code is layered. Tangled code → unreadable observability.
