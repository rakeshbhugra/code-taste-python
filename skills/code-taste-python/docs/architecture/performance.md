
# Performance & Benchmarking

← [Back to index](./README.md)

Performance work is its own discipline — bigger than testing, more contested. This doc is the orientation, not a deep dive. The actual practice you'll build by responding to real signals from real systems.

> **Companion doc:** this page covers the *external* tooling — profilers, tracers, load testers, monitors. For what to wire *into* your code (custom spans, health checks, metrics, pytest-benchmark), see [instrumentation.md](./instrumentation.md).

## The goal (the only one)

> **Know whether your system is fast enough to keep users happy, and have the tools to find out why when it isn't.**

Everything else is means to that end. Microbenchmarks, "we got a 100x speedup" blog posts, premature caching — none of it matters if the user-observable latency is fine. Conversely, if users are unhappy, no benchmark wins matter until you fix the actual problem.

## The mental model — 5 questions, 5 tools

Most engineers conflate all "performance work" into one bucket. It's actually five different questions with five different tools. Pick the right one for the question.

| Question | Tool | When |
|---|---|---|
| **"Is function X slow?"** | Profiler — `cProfile`, `py-spy`, `scalene`, `austin` | You suspect a specific function. Sample-based profilers like `py-spy` can attach to a running process: `py-spy top --pid 12345`. |
| **"Where is request latency going?"** | Distributed tracing — Logfire, OpenTelemetry, Datadog APM | Endpoint takes 800ms but you don't know where. Tracing breaks one request into spans (route → service → repo → DB → LLM). |
| **"Will it survive N concurrent users?"** | Load testing — Locust, k6, wrk, Gatling | Pre-launch capacity testing or proving an SLO. |
| **"Did the deploy regress?"** | APM + monitoring — Datadog, Logfire, Grafana, Sentry Performance | Continuous, in prod. p50/p95/p99 dashboards with alerts. |
| **"What's slowing my DB?"** | Query analysis — `EXPLAIN ANALYZE`, `pg_stat_statements`, slow query logs | Usually the answer for backend perf problems. ~80% of backend slowness is at the DB. |

Reach for the wrong one and you'll waste time. *"My endpoint is slow"* → start with **tracing** to see where time goes, not with a CPU profiler.

## The pragmatic principles (the load-bearing ones)

1. **Don't optimize without measuring.** "I bet this is slow" is wrong about 70% of the time. Measure first.
2. **Most slowness is I/O, not CPU.** In a typical Python backend: DB queries > network calls > serialization > Python execution. Optimize in that order.
3. **p99 matters more than p50.** A "fast on average" service can still have terrible tail latency. Track p99.
4. **Profile in prod (or prod-like) traffic, not dev.** Synthetic load doesn't have the same shape as real users.
5. **The cheapest perf win is almost always fixing N+1 queries.** ORMs make N+1 incredibly easy to write. Eager-load relationships.
6. **Set latency budgets early.** *"POST /lists must respond in <200ms p99."* Then alert on it. Without a budget, every regression is invisible.

## Load testing — three different shapes

"Load testing" is a family, not a single thing. Locust supports all three; you write the load *shape*, not just the rate.

| Shape | Pattern | What it tests |
|---|---|---|
| **Sustained** | 1x for 5–30 min | Baseline behavior at expected load. Most common. |
| **Spike** | 1x → 10x for 30 sec → 1x | Recovery from sudden surges (promos, viral moments). |
| **Soak** | 1x for hours or days | Resource leaks (memory, FDs, connections) that only appear over time. |
| **Ramp / hockey-stick** | 1x → 2x → 4x → 8x → break | Where's the cliff? Useful for capacity planning. |

Most teams only run sustained because it's the default in tutorials. **Sustained doesn't predict burst behavior, and it doesn't catch leaks.** If perf actually matters, you want at least one of each.

## "E2E + 3x sustained = ship it" — when it works, when it doesn't

This is a common heuristic. It's defensible for most apps. But know what it misses:

- **Tail latency** — "average is 200ms" can hide "p99 is 5s"
- **Burst traffic** — sustained 3x ≠ burst 10x
- **Cold start / cache miss** — load tests warm everything up
- **DB growth** — performance at 100k rows ≠ at 100M
- **External dependency limits** — your API is fine but the LLM provider rate-limits at 2x
- **Connection pool exhaustion / async deadlocks** — surface only at sustained concurrency
- **Memory creep** — 5-min test doesn't catch a 6-hour leak
- **Concurrent state** — Locust hits different rows; real users compete for the same rows

The cheapest single upgrade to the "3x" practice: **report p99 latency and error rate**, not just "it didn't crash." That alone catches most of what naive load testing misses.

## The four failure modes that bite production

These are the patterns that reliably break apps even when load tests pass. Worth being deliberate about each.

### 1. Database growth

Performance at 100k rows ≠ performance at 100M. Same query, same code, falls off a cliff when an index plan flips because the table no longer fits in memory.

The practices that catch it:
- **`EXPLAIN ANALYZE` on every slow query**, periodically — the plan changes at scale
- **Synthetic seeded data at 10x current size** in staging — run top queries monthly
- **Watch indexes** — the one that helps at 100k can hurt at 100M
- **`pg_stat_statements`** to find your top queries by total time, not just per-call time

### 2. Resource leaks ("forgot to close stuff")

Single most common production fire in long-running services. DB connections not returned to pool. File handles. HTTP clients held open. Background tasks accumulating. Memory dicts growing unbounded.

The fixes are mostly architectural:
- **Context managers everywhere.** `with session:`, `with httpx.AsyncClient() as client:`. If a thing has `close()`, wrap it.
- **Connection pools with hard limits.** When you hit them, fail loud — don't queue silently.
- **Background tasks tracked explicitly.** No "fire and forget" `asyncio.create_task()` without a registry. Otherwise tasks leak and exceptions get swallowed.
- **In prod monitoring: alert on the *trend*, not just absolute values.** A connection count slowly rising over 24h is the early warning.

The architecture connection: **lifecycle ownership should be explicit.** If you can't say *"object X owns the close() of resource Y,"* it'll leak. Unit of Work exists partly for this — it owns the transaction lifecycle so services can't forget.

### 3. Spikes and hockey-stick growth

Real traffic is not uniform. Promos, launches, retries-storm, cron jobs all the same minute. A system that handles 3x sustained can collapse under 10x for 30 seconds.

What to add:
- A spike test in your load suite (not just sustained)
- Rate limiting at the edges so retry storms don't amplify themselves
- Backpressure (return 503, don't queue forever)

### 4. Prod monitoring vs. synthetic

Synthetic load tests answer *"can it handle X?"* — monitoring answers *"is it actually handling reality?"* Reality always does weird things synthetic can't predict.

The minimum useful prod observability stack:
- **APM with tracing** — Logfire / Datadog. Per-endpoint p50/p95/p99 + error rate.
- **Structured logs** with `request_id`, `user_id`, `trace_id` — pivot from "this user's request was slow" to the exact spans.
- **Resource gauges** — DB pool usage, open connections, memory, FD count.
- **Saturation alerts**, not just availability — *"p99 latency > 800ms for 5 min"* beats *"is_up == false."*

The gap most teams have isn't tools — it's **someone actually looking at the dashboards.** Monitoring you don't watch is theatre. Set alerts, route them to a channel, page on real outages.

## The architecture connection

Clean layering makes performance work *dramatically* easier. With ports and adapters, your traces look like:

```
POST /lists                       820ms
├─ services.create_list           810ms
│  ├─ uow.lists.add               12ms
│  ├─ summarizer.summarize        780ms   ← obvious where time went
│  └─ uow.commit                  18ms
```

With tangled code, your traces look like one big blob and you have no idea what's slow. **Architecture is observability.** If your spans aren't readable, your code isn't either.

## What NOT to do (the traps)

1. **Microbenchmark obsession.** *"Is `dict` faster than `OrderedDict`?"* Almost never matters in a real backend. Real perf wins are at the DB or in algorithm choice, not in dict choice.
2. **"Scaling for the future" architecture.** Don't add Redis caching, message queues, or read replicas until you have measured load that justifies them. Premature scaling adds more bugs than it removes.
3. **Caching without measuring hit rate.** A cache with 5% hit rate is making things slower (extra hop on miss, extra memory). Always measure hit rate.
4. **Over-indexing on benchmark theater.** Companies post "we got 100x speedup" blogs. Cool. Doesn't apply to your app. Run your own measurements.
5. **Optimizing the hot path nobody hits.** Profile by *real traffic*. The route handling 90% of requests is where to focus.
6. **Adding observability you never look at.** Setting up Datadog and never opening it = paying for tools that don't help. Set alerts, or don't bother.

## The pragmatic upgrade path

You don't need to do all of this on day one. Add rigor when you see signals.

| Signal | What to add |
|---|---|
| Just shipping an MVP | E2E + sustained 3x load. Track p99 + error rate. |
| Users complain about occasional slowness while monitoring looks fine | p99 dashboards (not p50) |
| Promo or launch traffic crashes things | Spike load testing |
| First request after deploy is slow | Warm-up / pre-cache |
| Slowness creeps in after a week of uptime | Soak tests + memory profiling |
| Queries that were fast last month are slow now | Periodic `EXPLAIN ANALYZE` on top queries |
| Mysterious timeouts under moderate load | Connection pool tuning + tracing |
| "How will we handle 10x growth?" | Hockey-stick load tests + capacity plan |

## Tools landscape (Python-specific)

The actual things you'd reach for:

| Job | Tool | Notes |
|---|---|---|
| **Sample profiling, in-process** | `cProfile` | Built in. Outputs `.prof`, view with `snakeviz`. |
| **Sample profiling, attach to running proc** | `py-spy` | `py-spy top --pid` — works on prod, no code changes. The MVP. |
| **CPU + memory together** | `scalene` | Modern, shows GPU/memory too. Heavier. |
| **Async-aware profiling** | `austin` | Async-friendly sampler. |
| **Load testing** | Locust | Python-native, easy to write scenarios. You already use this. |
| **Tracing / APM** | Logfire | Pydantic team's product, pairs well with FastAPI. Already in orqestrator. |
| **Tracing — open standard** | OpenTelemetry | If you ever want to move providers. |
| **Error + perf monitoring** | Sentry | Performance plus exception tracking. |
| **DB analysis** | `EXPLAIN ANALYZE`, `pg_stat_statements` | Postgres built-ins. Worth more than any third-party tool. |

## Self-check

Try answering in your own words:

1. Your endpoint takes 800ms. What tool do you reach for first — profiler, tracer, or load test?
2. Why does "average latency is fine" not mean "users are happy"?
3. A team caches every DB query and reports a 30% speedup. What number would change your mind about whether the cache earns its keep?
4. Why does a 5-minute Locust test miss memory leaks?
5. Architecture and performance — what's the connection?

## Things to steal

- **Five tools, five questions.** Don't reach for a profiler when you need a tracer (or vice versa).
- **p99, not p50.** Average lies; tails tell the truth.
- **Load shapes are a family** — sustained, spike, soak, ramp. Most teams only do sustained.
- **The cheapest perf upgrade for any project: report p99 + error rate, not just "didn't crash."**
- **Most slowness is at the DB.** Learn `EXPLAIN ANALYZE` before any other perf tool.
- **Resource leaks are an architecture problem, not a tuning problem.** Lifecycle ownership must be explicit.
- **Monitoring you don't watch is theatre.** Set alerts, or don't bother.
- **Architecture is observability.** Clean spans come from clean layers.
- **Don't optimize without measuring. Don't measure without acting on it.**
