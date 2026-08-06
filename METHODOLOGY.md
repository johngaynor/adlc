# The ADLC Methodology

> **This is the spine.** Every template and every skill in this plugin quotes the
> vocabulary defined here. Change it here first, then propagate — never redefine
> these concepts inline in a skill.

ADLC (AI Development Life Cycle) is a small, portable set of conventions that make
an AI coding agent behave like a disciplined senior engineer on *your* codebase:
it knows where code goes, what it's allowed to do without asking, and how to prove
its work. It is framework-agnostic — it works on a Next.js monorepo, a Python
service, or a React Native app equally.

The methodology is five ideas. Adopt them in order; each is useful on its own.

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
add. The spec lives on the task's card in the PM system — the `# Specification`
block of its issue, written by the lifecycle skills through the PM seam
(`reference/pm-seam.md`) — never in repo-local spec files. The issue is the single
source of truth while the task is in flight and its durable record after: when
the PR merges, the card simply closes — the repo carries the merged code and its
PR description, not a duplicate write-up. Issue-level artifacts are
engineering-level: product direction and product decisions live upstream at the
project/milestone, not on the card. Small fixes skip this — just fix them.

**The lessons loop.** Corrections are captured, not lost. When the agent is
corrected on something that will recur, it appends a structured entry to
`.claude/lessons.md`:

```
## <short title>
**Context**: where this came up.
**Problem**: what went wrong.
**Rule**: the durable rule that prevents it.
**Applies to**: the paths/areas this governs.
```

Over sessions, `lessons.md` becomes a codebase-specific memory that makes the
harness measurably smarter. It is reviewed at session start.

Lessons mature along a graduation ladder — **correction → lesson → rule → skill**:
a recurring lesson becomes a `CLAUDE.md` rule, and one that describes a whole
repeatable workflow becomes a project skill (see § Extending the harness with
project skills).

---

## 5. Isolated parallel work

The unit of parallelism: **one task = one isolated workspace = one PR.** An agent
never edits the main checkout — it stays on the default branch as the stable
reference copy. Every PR-bound task, however small, happens in its own isolated
workspace (a git worktree or equivalent) and integrates through a PR a human
reviews and merges. With no shared mutable state before merge, any number of
agents can work in parallel.

The rule is the **invariant, not the mechanism** — detection over configuration:

1. **Detect.** Before starting PR-bound work, check whether you are already
   isolated: if `git rev-parse --git-dir` and `git rev-parse --git-common-dir`
   differ, you are in a linked worktree — proceed, create nothing. (Guard: if
   `git rev-parse --show-superproject-working-tree` prints a path, you are in a
   submodule, not a worktree.) Platforms with built-in worktrees (e.g.
   Conductor) pass this check automatically — zero configuration.
2. **Provision natively.** Not isolated? Use the harness's native worktree
   tool if one exists (e.g. Claude Code's built-in worktree support).
3. **Fall back to git.** Otherwise: `git worktree add .worktrees/<task-slug>`,
   with `.worktrees/` in `.gitignore`.

**Lifecycle:** implement and validate inside the workspace; ship a PR from it;
once the PR is pushed the workspace is disposable — the branch lives on the
remote. At ship time, machine-generated branch names (e.g. `worktree-*`) are
renamed to proper feature branches; sensible platform-chosen names are kept.

---

## Extending the harness with project skills

The plugin's own skills (`adlc:*`) ship with the plugin and are never modified
by consuming repos. A repo can add its *own* skills — committed, team-shared
workflows in the open agent-skills format (YAML frontmatter with `name` and a
trigger-style `description`, then an imperative body).

**TBD(project-skills):** the convention for where those skills live and how
harnesses discover them (canonical home, discovery wiring, docs pointer,
ignore rules) is parked while it is redesigned. Until then, `/adlc:init`
scaffolds nothing for it and `/adlc:add-skill` asks the user where to write.

---

## The three core principles

- **Simplicity first** — make every change as simple as possible; touch minimal code.
- **No laziness** — find root causes, no temporary patches, senior-engineer standard.
- **Minimal impact** — changes touch only what's necessary; don't introduce risk.
