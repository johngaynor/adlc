---
name: init
description: Use when a repo has no ADLC harness yet (no root CLAUDE.md, or the user asks to "set up ADLC", "initialize the harness", "add the AI conventions"). Scaffolds an opinionated CLAUDE.md with a Task Router and boundary-labeled rules, a lessons.md, and a specs directory — tailored to the project's detected stack.
---

# Initialize the ADLC harness

Scaffold this project with the ADLC engineering harness: an opinionated root
`CLAUDE.md` (Task Router + Always/Ask-First/Never/Validation), a `.claude/lessons.md`
self-improvement log, and a `.ai/specs/` directory. Read
[`METHODOLOGY.md`](../../METHODOLOGY.md) and [`CONVENTIONS.md`](../../CONVENTIONS.md)
if you need the vocabulary — do not redefine it here.

This skill is **opinionated**: it drops a complete, working `CLAUDE.md` with a real
Task Router seeded for the detected stack, not an empty skeleton. The user grows it
from a strong starting point.

## Workflow

### 1. Detect the project

Inspect the repo to fill the template accurately. Do not guess — read files:

- **Stack & language**: `package.json` (Node/TS), `pyproject.toml`/`requirements.txt`
  (Python), `go.mod` (Go), `Cargo.toml` (Rust), `Gemfile` (Ruby), mobile markers
  (`app.json`, `Podfile`), etc.
- **Package manager**: lockfile — `yarn.lock`, `pnpm-lock.yaml`, `package-lock.json`,
  `bun.lockb`, `poetry.lock`, `uv.lock`.
- **Validation commands**: read `scripts` in `package.json` (or `Makefile`,
  `justfile`, `pyproject`) to find the *real* build / typecheck / lint / test
  commands. These become the `Validation Commands` block — never invent them.
- **Structure**: monorepo (workspaces / `packages/*` / `apps/*`) vs single package.
  A monorepo gets Task Router rows and a note to add nested `CLAUDE.md` files.
- **Existing conventions**: a `README`, `CONTRIBUTING.md`, or existing docs that
  reveal naming/architecture rules worth seeding into the Task Router.

### 2. Confirm the essentials

If detection is ambiguous, ask the user **concise** questions — project name, the
one-line "what is this", and the canonical build/test commands. If detection is
confident, state what you found and proceed.

### 3. Check for conflicts (idempotency)

- If a root `CLAUDE.md` already exists, do **not** overwrite it. Show the user the
  diff between what you'd generate and what's there, and offer to merge the Task
  Router / boundary sections in instead. Ask first.
- If `.claude/lessons.md` or `.ai/specs/` already exist, leave them and only add
  what's missing.

### 4. Write the files

From the plugin's templates (available at `${CLAUDE_PLUGIN_ROOT}/templates/` — the
plugin's install directory; reproduce the structure inline if that path is not
resolvable at runtime), render and write:

- **`CLAUDE.md`** (repo root) — from `templates/CLAUDE.md.template`, with every
  `{{PLACEHOLDER}}` filled from step 1. The Task Router must have real rows for
  this project (e.g. "Add an API route → …", "Add a UI screen → …", "Write a
  migration → …"), not the generic examples. Seed 4–8 rows that match the stack.
- **`.claude/lessons.md`** — from `templates/lessons.md.template` (empty log with
  the entry format documented at the top).
- **`.ai/specs/README.md`** — from `templates/spec.md.template`'s sibling note, or
  a short pointer explaining the `{YYYY-MM-DD}-{kebab-title}.md` convention. Create
  the `.ai/specs/` directory.

### 5. Report and hand off

Summarize what you created and tell the user the immediate next moves:

- Review `CLAUDE.md` — especially the Task Router rows and Validation Commands.
- The harness ships more skills: `/adlc:write-spec`, `/adlc:ship-pr`,
  `/adlc:add-lesson`. Mention they're available.
- Suggest committing the scaffold.

## Boundaries

- **Never** overwrite an existing root `CLAUDE.md` without explicit confirmation.
- **Ask First** before inventing validation commands you could not find — a wrong
  build command is worse than an empty one.
- **Always** fill placeholders from real detection; leave a `TODO(adlc)` marker
  where you genuinely could not determine a value, so it's greppable.

## Done when

`CLAUDE.md`, `.claude/lessons.md`, and `.ai/specs/` exist, the Task Router and
Validation Commands reflect this specific project, and you've told the user what to
review.
