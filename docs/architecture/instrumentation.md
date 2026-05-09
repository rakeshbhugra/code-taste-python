
# Code-Level Instrumentation

← [Back to index](./README.md)

What to wire *into* your code for performance measurement, profiling, and benchmarking. Companion to [performance.md](./performance.md), which covers the *external* tooling.

## The goal (the only one)

> **Auto-instrumentation tells you which DB query was slow. Code-level instrumentation tells you which *workflow* was slow. You want both, but most teams stop after auto-instrumentation and miss the second half.**

The right amount of instrumentation: enough that a slow request's trace reads like a story ("user uploaded → service summarized → DB saved → notification sent"), not so much that you're logging every variable assignment.

## The default rule — don't instrument what's already free

Before adding any span, metric, or timer, check if you're getting it for free. Most teams accidentally re-implement what auto-instrumentation already gives them.

| What | Already free if you have… | Add code? |
|---|---|---|
| HTTP request duration | FastAPI middleware (Logfire/OTel) | No |
| Per-DB-query timing | SQLAlchemy auto-instrumentation | No |
| Per-LLM-call timing | Logfire's OpenAI/Anthropic instrumentation | No |
| HTTP client calls (httpx, requests) | OTel `httpx-instrumentation` | No |
| Function-level CPU profiling | `py-spy --pid` | No |
| Memory profiling | `scalene`, `py-spy --memory` | No |

**Don't add a `with span("query_user")` around a SQLAlchemy call.** The ORM already emits that span. You're duplicating data and making traces noisier.

## The five things worth wiring in

In rough priority order. Add 1-2 first, the rest only when you have a specific reason.

### 1. Custom spans for multi-step operations

The single highest-leverage code change. Auto-instrumentation breaks down "this request" into individual primitives. *You* know the workflow they belong to.

```python
import logfire

def summarize_list(list_id: str, repo, summarizer):
    with logfire.span("summarize_list", list_id=list_id):
        lst = repo.get(list_id)
        if lst is None:
            raise ListNotFound()
        return summarizer.summarize(SummaryRequest(text=lst.contents_as_text()))
```

Without the span, the trace shows: SELECT, then LLM call, then maybe an UPDATE. With the span, you see them grouped under one bar labeled `summarize_list` with `list_id` attached. Pivoting from "which list took 5s" to "which list_id is slow" is one click.

**Where to add spans:**
- Service methods (`create_list`, `share_list`, `summarize_list`)
- Multi-step workflows (a background job with several phases)
- Anything that calls 2+ adapters in sequence

**Where NOT to add spans:**
- Inside loops (span explosion, kills tracing performance)
- Pure in-memory calculations (no I/O, fast — not worth a span)
- Single-primitive operations already auto-instrumented

The good rule: **spans should match service-method names.** One per use case, attributes for the IDs that uniquely identify the call.

### 2. A real `/health` endpoint

A `/health` that returns `200 OK` unconditionally is theatre. A real one checks the system's *actual* dependencies and exposes per-dependency status.

```python
@router.get("/health")
async def health(uow: UnitOfWork = Depends(get_uow)):
    checks = {}

    try:
        async with uow:
            await uow.session.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"down: {type(e).__name__}"

    try:
        await summarizer.ping()  # if the adapter has a cheap health probe
        checks["summarizer"] = "ok"
    except Exception as e:
        checks["summarizer"] = f"down: {type(e).__name__}"

    overall = "ok" if all(v == "ok" for v in checks.values()) else "degraded"
    status_code = 200 if overall == "ok" else 503
    return JSONResponse({"status": overall, "checks": checks}, status_code=status_code)
```

Two endpoints if you want to be precise:
- `/health/live` — am I running? (always 200 unless the process is broken)
- `/health/ready` — am I ready to serve traffic? (checks DB, deps, etc.)

K8s, load balancers, and uptime monitors can each hit the right one. Without depth, your "is it up?" page lies during real outages — the process is up but the DB is gone.

### 3. Custom metrics — only when you want a dashboard

Counters, histograms, gauges. Reach for these only when you've identified a specific dashboard or alert you want.

```python
import logfire

# Counter — count of things that happened
lists_created_counter = logfire.metric_counter("lists_created_total")

# Histogram — distribution of values
summarization_tokens = logfire.metric_histogram("summarization_tokens_used")

# Gauge — current value of something
db_pool_in_use = logfire.metric_gauge("db_pool_connections_in_use")

def create_list(...):
    # ...
    lists_created_counter.add(1)

def summarize_list(...):
    result = summarizer.summarize(...)
    summarization_tokens.record(result.tokens_used)
    return result
```

Use cases:
- **Counters** — business events you want a graph of (signups, payments, errors)
- **Histograms** — distributions you want percentiles for (LLM tokens, request size, queue depth)
- **Gauges** — point-in-time values (pool sizes, queue length, active sessions)

**Skip this until you actually want a dashboard for it.** Most teams over-add metrics, then never look at them. Span data + structured logs cover 90% of what dashboards need.

### 4. pytest-benchmark for hot functions

For *micro*-benchmarks of specific functions where regressions matter — a parser, a hashing routine, an expensive computation. 99% of backend code doesn't need this.

```python
# tests/perf/test_slug_perf.py
def test_slug_generation_speed(benchmark):
    benchmark(List.slug_from_name, "Some Long List Name With Many Words")
```

Run with `pytest tests/perf/`. pytest-benchmark records mean/median/stddev across runs and can fail CI if a function gets >X% slower.

When to use:
- You have a function on a hot path under suspicion
- You want regression-tracking over time

When not to:
- General "is my app fast?" — that's a load test, not a microbenchmark
- Code that does I/O — pytest-benchmark times wall-clock, I/O variance dominates

### 5. Latency budget assertions in tests

```python
@pytest.mark.asyncio
async def test_create_list_meets_p99_budget(client):
    durations = []
    for _ in range(50):
        start = time.perf_counter()
        await client.post("/lists", json={"name": "Test"})
        durations.append(time.perf_counter() - start)
    p99 = sorted(durations)[int(len(durations) * 0.99)]
    assert p99 < 0.2, f"p99 was {p99:.3f}s, budget is 200ms"
```

This is rare and arguably brittle (test machine speed varies), but it catches a class of regression that nothing else does: *"this got 5x slower and nobody noticed."* Reach for it only on flows where latency is part of the contract (auth, search, anything user-facing).

## Worked example — instrumenting a service end-to-end

Putting it together for hypedar's `share_list`:

```python
import logfire
import structlog

logger = structlog.get_logger()
shares_counter = logfire.metric_counter("lists_shared_total")

def share_list(list_id: str, owner_id: str, recipient_email: str, uow: UnitOfWork) -> None:
    with logfire.span("share_list", list_id=list_id, owner_id=owner_id):
        with uow:
            lst = uow.lists.get(list_id)
            recipient = uow.users.get_by_email(recipient_email)

            if lst is None or recipient is None:
                logger.info("share_list_not_found", list_id=list_id, recipient_email=recipient_email)
                raise NotFound()

            lst.share_with(recipient.id, actor_id=owner_id)
            uow.lists.add(lst)
            uow.commit()

        shares_counter.add(1)
        logger.info("list_shared", list_id=list_id, owner_id=owner_id, recipient_id=recipient.id)
```

What this gives you:
- A trace span `share_list` with the IDs as attributes
- Sub-spans (auto): SELECT lst, SELECT user, UPDATE lst, COMMIT
- A counter that ticks every successful share
- A structured log line with all the relevant fields, tied by `request_id` (from the correlation-id middleware)
- A second log line for the not-found case so you can graph share-failures

Total added code: ~3 lines beyond the business logic. That's the right ratio.

## The traps

### Trap 1: span explosion

```python
for item in items:
    with logfire.span("process_item"):    # ← 1000 items = 1000 spans
        process(item)
```

Don't span inside loops. Span outside, log inside if needed:

```python
with logfire.span("process_items", count=len(items)):
    for item in items:
        process(item)
```

### Trap 2: spans around already-instrumented calls

```python
with logfire.span("get_user"):                       # ← Logfire SQLAlchemy already instruments this
    user = session.query(User).get(user_id)
```

You've doubled the trace data and added latency for no information. Skip the wrapper.

### Trap 3: metrics that nobody dashboards

```python
foo_counter = logfire.metric_counter("foo_things_happened")
foo_counter.add(1)   # ← exists in code, never queried
```

Cardinality of unread metrics is technical debt. Add a metric only when you've named the dashboard or alert it'll feed.

### Trap 4: metrics with high-cardinality labels

```python
counter.add(1, labels={"user_id": user_id})   # ← unique per user
```

Each label value creates a separate time series. Per-user labels mean millions of time series, which crashes your monitoring backend's bill. Use:
- Low-cardinality labels (status, plan_tier, region) on metrics
- High-cardinality fields (user_id, list_id) on **logs and span attributes**, not metric labels

### Trap 5: timing decorators that pretend to be tracing

```python
@time_it
def some_function(): ...
```

Custom timing decorators that print or log durations are a sign you're missing real tracing. Logfire/OTel give you all this for free if you've added them. The exception: very narrow, in-memory hot paths where you want pytest-benchmark history.

### Trap 6: instrumenting before measuring

Adding spans speculatively *all over the codebase* before knowing what's actually slow is busywork. Add spans where:
- A user-reported "this is slow" report came in
- Your traces are sparse around a specific operation
- You're about to refactor and want a baseline

Don't pre-emptively span every function. Trace data has cost.

## How to spot where spans would help

Open Logfire (or wherever your traces live), pick the slowest 10% of requests, look at the trace. Ask:

| Symptom | What to add |
|---|---|
| The trace is one giant flat list of DB calls and LLM calls | Wrap the workflow in a span |
| You see total time but not which sub-step is slow | Add child spans to the workflow |
| You can tell the LLM call was slow but not for which list | Add `list_id` as an attribute to the span |
| `/health` returns 200 during a partial outage | Add depth to the health check |
| You want to graph "lists created per hour" | Add a counter |
| One function is microbenchmark-suspect | pytest-benchmark |

## Pragmatic order

For a new project: add nothing yet. Logfire's auto-instrumentation + your request-logging middleware is enough at the start.

When the app grows:

| Stage | What to add |
|---|---|
| 1 | Custom span around each service method (3–5 lines per service) |
| 2 | A real `/health` endpoint with depth |
| 3 | Logfire counters for top business events (signups, payments) |
| 4 | Latency budget assertions on user-critical flows |
| 5 | pytest-benchmark for any hot function that's been a regression source |

Don't do all five on day one. Stages 1 and 2 cover most of the value.

For **qhive** today: nothing yet. The app is one endpoint.
For **hypedar** during refactor: stage 1 around services + stage 2 health endpoint.
For **orqestrator**: it already has Logfire — stage 1 for any service-method that runs a non-trivial workflow.

## Self-check

1. Why is `with span("query_user"):` around a SQLAlchemy call usually wrong?
2. What's the difference between a counter, a histogram, and a gauge — and when do you use each?
3. Why is per-user labeling on a metric usually a mistake?
4. Your trace shows the request took 800ms but you can't tell where the time went. What's missing?
5. When is pytest-benchmark the right tool, and when is it the wrong tool?

## Things to steal

- **Default to nothing. Auto-instrumentation handles 80%.** Add code only when you've identified a gap.
- **One span per service method.** That's the high-leverage shape — workflows readable, not noisy.
- **Span attributes, not metric labels, for high-cardinality fields.** `list_id` goes on a span, not a counter.
- **Real health checks have depth.** `/health` should ping the things you actually depend on, or it's lying.
- **Don't span inside loops.** Span outside, count the iterations as an attribute.
- **Don't add metrics until you've named the dashboard.** Unread metrics rot.
- **pytest-benchmark for micro-CPU regressions only.** Not a substitute for load tests.
- **The right ratio: ~3 lines of instrumentation per service method.** More and you're noise; less and traces aren't readable.
