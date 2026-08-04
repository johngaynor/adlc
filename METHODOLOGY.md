# The ADLC Methodology

> **This is the spine.** Every template and every skill in this plugin quotes the
> vocabulary defined here. Change it here first, then propagate — never redefine
> these concepts inline in a skill.

ADLC (AI Development Life Cycle) is a small, portable set of conventions that make
an AI coding agent behave like a disciplined senior engineer on *your* codebase:
it knows where code goes, what it's allowed to do without asking, and how to prove
its work. It is framework-agnostic — it works on a Next.js monorepo, a Python
service, or a React Native app equally.

The methodology is four ideas. Adopt them in order; each is useful on its own.

---

## 1. Layered instruction files, discovered by proximity

One `CLAUDE.md` at the repo root holds global rules. As the codebase grows, split
local architecture into nested `CLAUDE.md` (or `AGENTS.md`) files colocated with
the code they govern — one per package, service, or module. The agent reads the
root always, and the nearby ones when it works in that area.

**Rule of thumb:** if a rule only matters inside `services/billing/`, it belongs in
`services/billing/CLAUDE.md`, not the root. Keep the root file about the *whole*
project.

---

## 2. The Task Router

At the top of the root `CLAUDE.md`, keep a dispatch table: **task → guide**.

```
| Task | Guide |
|------|-------|
| Adding an API endpoint | docs/api-conventions.md |
| Building a UI screen    | docs/ui-guide.md + design-system.md |
| Writing a migration     | docs/db.md |
```

Before doing research or writing code, the agent matches its task to one or more
rows and reads *those* guides first. This converts a giant unread instruction file
into **lookup-on-demand** — the single highest-leverage habit in ADLC. A task often
matches multiple rows; all matching guides apply.

---

## 3. Boundary labels: Always / Ask First / Never / Validation

Every rule is filed under exactly one of four headings, so the agent has
unambiguous authority levels instead of prose it must interpret:

- **Always** — required defaults the agent applies without asking.
- **Ask First** — decisions that need a human before changing behavior, scope,
  dependencies, public contracts, or architecture.
- **Never** — hard prohibitions and unsafe shortcuts.
- **Validation Commands** — the exact, real commands that prove a change works
  (build, typecheck, lint, test). The agent runs the smallest relevant set.

When you add a rule to a `CLAUDE.md`, always put it under one of these four.

---

## 4. Spec-first development + the lessons loop

Two habits that compound over time:

**Spec-first.** Non-trivial work (roughly 3+ steps or an architectural decision)
gets a short spec *before* code: what, why, phases, and the test coverage it will
add. Specs live in `.ai/specs/{YYYY-MM-DD}-{kebab-title}.md` and ship with the
repo, so the intent behind code is reproducible and reviewable. Small fixes skip
this — just fix them.

**The lessons loop.** Corrections are captured, not lost. When the agent is
corrected on something that will recur, it writes a structured entry as a new
file in `.claude/lessons/` — one file per lesson, so parallel branches never
merge-conflict on a shared log:

```
# <short title>
**Context**: where this came up.
**Problem**: what went wrong.
**Rule**: the durable rule that prevents it.
**Applies to**: the paths/areas this governs.
```

Over sessions, `.claude/lessons/` becomes a codebase-specific memory that makes
the harness measurably smarter. Every file there is reviewed at session start.

---

## The three core principles

- **Simplicity first** — make every change as simple as possible; touch minimal code.
- **No laziness** — find root causes, no temporary patches, senior-engineer standard.
- **Minimal impact** — changes touch only what's necessary; don't introduce risk.
