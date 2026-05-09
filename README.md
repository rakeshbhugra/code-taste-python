# code-taste-python

A layered-architecture playbook for Python backends — Domain / Repository / Service / Adapter / Unit-of-Work — distilled from *Architecture Patterns with Python* (Cosmic Python) and DDD into plain-English working notes.

It also ships as a [Claude Code Skill](https://code.claude.com/docs/en/skills) so Claude can review and structure code against these patterns automatically.

## What's in here

```
.
├── .claude-plugin/
│   └── marketplace.json                  # makes this installable as a plugin
├── skills/
│   └── code-taste-python/
│       ├── SKILL.md                      # the Claude Code skill entrypoint
│       └── docs/architecture/            # the playbook
│           ├── README.md                 # overview, dependency rule, vocabulary
│           ├── domain.md
│           ├── repository.md
│           ├── service.md
│           ├── adapter.md
│           ├── unit-of-work.md
│           ├── project-structure.md
│           ├── testing.md
│           ├── performance.md
│           ├── logging.md
│           └── instrumentation.md
└── books/                                # source material
```

Read the docs directly, or install the skill so Claude uses them on your behalf.

## Install as a Claude Code skill

Inside Claude Code:

```
/plugin marketplace add rakeshbhugra/code-taste-python
/plugin install code-taste-python@code-taste-python
/reload-plugins
```

Updates ship via `/plugin marketplace update code-taste-python` followed by `/reload-plugins`.

### Verify

In Claude Code, ask:

> review this file against the code-taste architecture

If the skill loaded, Claude will pull the relevant doc(s) from `skills/code-taste-python/docs/architecture/` and review against the dependency rule and layer traps.

## Use the docs without the skill

Just clone anywhere and read `skills/code-taste-python/docs/architecture/README.md` first — it's the entry point and explains the dependency rule and vocabulary used by the rest.

```bash
git clone https://github.com/rakeshbhugra/code-taste-python.git
```
