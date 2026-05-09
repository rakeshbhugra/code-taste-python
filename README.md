# code-taste-python

A layered-architecture playbook for Python backends — Domain / Repository / Service / Adapter / Unit-of-Work — distilled from *Architecture Patterns with Python* (Cosmic Python) and DDD into plain-English working notes.

It also ships as a [Claude Code Skill](https://code.claude.com/docs/en/skills) so Claude can review and structure code against these patterns automatically.

## What's in here

```
.
├── SKILL.md                       # the Claude Code skill entrypoint
├── .claude-plugin/
│   └── marketplace.json           # makes this installable as a plugin
├── docs/
│   └── architecture/              # the playbook
│       ├── README.md              # overview, dependency rule, vocabulary
│       ├── domain.md
│       ├── repository.md
│       ├── service.md
│       ├── adapter.md
│       ├── unit-of-work.md
│       ├── project-structure.md
│       ├── testing.md
│       ├── performance.md
│       ├── logging.md
│       └── instrumentation.md
└── books/                         # source material
```

Read the docs directly, or install the skill so Claude uses them on your behalf.

## Install as a Claude Code skill

### Option 1 — Manual git clone

```bash
git clone https://github.com/rakeshbhugra/code_taste.git ~/.claude/skills/code-taste-python
```

Claude Code auto-discovers skills in `~/.claude/skills/`. The SKILL.md references `./docs/architecture/` relatively, so no further setup is needed.

Update with `git pull` in that directory; uninstall by removing it.

### Option 2 — Plugin marketplace (TODO, this is WIP please go with the option 1)

Inside Claude Code:

```
/plugin marketplace add rakeshbhugra/code_taste
/plugin install code-taste-python@code-taste-python
```

Updates ship via `/plugin update`.

### Verify

In Claude Code, ask:

> review this file against the code-taste architecture

If the skill loaded, Claude will pull the relevant doc(s) from `docs/architecture/` and review against the dependency rule and layer traps.

## Use the docs without the skill

Just clone anywhere and read `docs/architecture/README.md` first — it's the entry point and explains the dependency rule and vocabulary used by the rest.

```bash
git clone https://github.com/rakeshbhugra/code_taste.git
```
