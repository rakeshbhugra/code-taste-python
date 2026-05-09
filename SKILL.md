---
name: code-taste-python
description: Review and structure Python backend code against the layered-architecture playbook in ./docs/architecture/. Use when asked to review architecture, structure a new feature, check whether code follows the patterns (Domain/Repository/Service/Adapter/UoW), or audit what's missing in code quality / tooling / tests / observability.
---

# Code Taste (Python)

Code-structure review and guidance, grounded in the architecture playbook at `./docs/architecture/` (sibling to this file).

The docs define the vocabulary. Don't re-explain concepts they cover — link to the file or quote the relevant part.

## Step 1 — Load only the relevant docs

The full doc set is at `./docs/architecture/`:

| File | Use when… |
|---|---|
| `README.md` | Need the overview / dependency rule / vocabulary cheat-sheet |
| `domain.md` | Reviewing entities, business rules, anemic-vs-rich questions |
| `repository.md` | Reviewing data access, swap-cases, fakes for tests |
| `service.md` | Reviewing use-case orchestration; "rules in service" smells |
| `adapter.md` | Any external system seam (auth, LLM, payment, email, file storage) |
| `unit-of-work.md` | Multi-repo writes, transaction boundaries, atomicity |
| `project-structure.md` | New project setup, directory layout, dependency-rule enforcement |
| `testing.md` | Test pyramid, doubles vs mocks, what to test where |
| `performance.md` | Profiling vs tracing vs load testing, prod failure modes |
| `logging.md` | Structured logging, log levels, what to log where |
| `instrumentation.md` | Metrics, traces, observability seams |

**Don't load all of them. Load the 1–3 that match the task.** For a service review: Domain + Service + UoW. For new project setup: Project Structure + Testing. For "what am I missing?": README (vocab/dep rule) + the doc closest to the gap.

## Step 2 — Match task to review mode

**"Review this code":**
- Identify what layer(s) the code is in (domain / service / adapter / route)
- Check that layer's "Traps" section from its doc
- Verify the dependency rule (does anything inward import outward?)
- Run the "How to spot it in your code" check from each relevant doc

**"Structure this new feature":**
- Sketch the directory layout from `project-structure.md`
- Sketch the port + adapter shape if external systems are involved
- Identify the right test boundary (domain unit / service / integration / e2e)
- Don't over-layer if the feature is small — match rigor to scope

**"What am I missing?":**
- Compare against a senior tooling baseline (ruff + mypy strict + pytest with coverage ratchet + pre-commit + CI)
- List concrete additions in priority order with one-line justifications
- Surface the top 3–5 gaps with the highest impact, not an exhaustive audit

## Step 3 — Apply core principles

These are the durable principles behind the playbook:

- **Concept-first, plain English.** The docs are the vocabulary; reference them, don't re-derive.
- **Calibrated honesty over flattery.** Tell the reader what's wrong, not what's "great with minor improvements."
- **Rent test on every abstraction.** "Could you name a real second implementation?" If not, the abstraction is speculative — say so.
- **Match rigor to project stage.** Premature layering, premature scaling, premature optimization — all wrong. Stage-appropriate is the goal.
- **Tests are the fitness function.** When suggesting refactors, suggest the test that would prove it works.

## Output format

Default to this shape, kept tight:

1. **What's right** — 1–3 bullets, brief. No flattery padding.
2. **What's wrong / missing** — numbered. Each item: the issue, the layer it belongs to, the doc reference, the fix shape.
3. **Concrete next steps** — priority-ordered, with rough effort estimate.

Match length to task. A simple "is this right?" gets a 5-line answer. A full code review gets the structured format above.

## What NOT to do

- **Don't re-teach the architecture.** Reference the docs.
- **Don't suggest patterns without the rent test.** Speculative abstractions are smells.
- **Don't recommend exhaustive refactors on early-stage code.** Lift hygiene tooling first; layering can wait.
- **Don't add abstractions for hypothetical futures** ("you might want to swap…"). Real second case or skip it.
- **Don't gold-plate.** Match rigor to project stage.
- **Don't be exhaustive.** Surface the top issues, not every issue.

## Quick reference — the dependency rule (load-bearing)

Memorize this without reading the docs:

```
domain      ←  imports nothing external
ports       ←  imports domain only
adapters    ←  imports domain, ports, external SDKs
services    ←  imports domain, ports
entrypoints ←  imports services, schemas
bootstrap   ←  imports everything (the wiring)
```

Arrows point inward. A `grep` for forbidden imports is enough to enforce. If `domain/` imports from `adapters/`, the architecture is broken.
