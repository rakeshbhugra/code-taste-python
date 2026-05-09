# Learning Good Code — Reading Plan

Goal: build a real taste model for what "good code" looks like by reading three carefully chosen repos in order. Don't skim — read actively.

## The three repos, in order

### 1. `requests` (Python) — start here
**Why:** small, focused, famously readable. The canonical "good Python library." Pure code-quality lessons with no framework noise.

**Reading order (don't deviate):**
1. `src/requests/api.py` — the public surface. Deceptively simple; study how the API is shaped.
2. `src/requests/sessions.py` — the real engine. This is where the design choices live.
3. `src/requests/models.py` — `Request` / `Response` data classes done right.
4. *(skip on first pass)* `adapters.py`, `auth.py`

**Time budget:** 1–2 weeks of evening reading.

---

### 2. `httpx` (Python) — optional, ~3 days
**Why:** modern reimagining of `requests` with async + types. Same problem, ~10 years later, different idioms. Useful contrast — and you'll use it as an AI engineer anyway.

**Lens:** compare every design decision back to `requests`. What changed? Why?

---

### 3. `langgraph` — read with a specific lens
**Why:** you use it daily, so the domain is free. But it's a fast-moving framework with heavy abstraction — read it knowing you're studying a framework, not a model of clean code.

**Reading order (don't read top-to-bottom — that way lies madness):**
1. `langgraph/graph/state.py` — core state abstraction
2. `langgraph/graph/graph.py` — graph construction API
3. `langgraph/pregel/` — the execution engine. **This is the interesting part.**
4. *(skip)* integrations, prebuilt agents — surface area, not lessons

---

## Active reading technique (do this for every repo)

Keep a notes file per repo with **two columns**:

| Things I'd steal | Things I'd do differently |
| --- | --- |
| ... | ... |

Forces active reading. After all three repos you'll have a real, defensible taste.

## Rules

- No top-to-bottom reading. Follow the order above.
- One file at a time. Finish it before moving on.
- If you don't understand something, write the question down — don't skip silently.
- Don't read the tests first. Read the code, predict the behavior, then check tests to verify.
