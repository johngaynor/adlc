---
name: init
description: Use when a repo has no ADLC harness yet (no root CLAUDE.md, or the user asks to "set up ADLC", "initialize the harness", "add the AI conventions"). Scaffolds an opinionated CLAUDE.md with a Task Router and boundary-labeled rules, plus a lessons.md — tailored to the project's detected stack.
---

# Initialize the ADLC harness

Scaffold this project with the ADLC engineering harness: an opinionated root
`CLAUDE.md` (Task Router + Always/Ask-First/Never/Validation) and a
`.claude/lessons.md` self-improvement log. Read
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
- If `.claude/lessons.md` already exists, leave it and only add what's missing.
- Project-skills convention (step 4): if `.claude/skills` is already a symlink to
  `../.ai/skills`, the whole scaffold is a no-op. If `.claude/skills` exists as a
  **real directory** with content, do not touch it silently — offer to move its
  contents into `.ai/skills/` and replace the directory with the symlink, and ask
  first. If `.ai/skills/` already exists, leave its contents untouched and add
  only what's missing (e.g. a missing `README.md` or `AGENTS.md` pointer).

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
- **`.gitignore`** — ensure a `.worktrees/` entry exists (append to the existing
  file, or create it), so fallback worktrees (METHODOLOGY.md idea 5) are never
  committed.
- **Project skills scaffold** — the tool-neutral home for this repo's own agent
  skills (see METHODOLOGY.md § "Extending the harness with project skills"):
  - **`.ai/skills/README.md`** — from `templates/project-skills-README.md.template`.
    This seed file is load-bearing: git doesn't track empty directories, so
    without it a fresh clone would leave the symlink below dangling.
  - **`.claude/skills`** — a *relative* symlink to the canonical directory:
    `ln -s ../.ai/skills .claude/skills`. Commit the symlink itself; it is what
    gives Claude Code native discovery of project skills (session skill list +
    `/<name>` invocation). **Windows caveat:** symlink creation needs Developer
    Mode or `core.symlinks=true`. If creation fails, fall back to keeping
    `.claude/skills/` as a real directory and tell the user plainly that it will
    not auto-track `.ai/skills/` — don't half-fix it silently.
  - **`AGENTS.md`** — ensure a short "Project skills" section pointing other
    (non-Claude) harnesses at `.ai/skills/<name>/SKILL.md`. Create the file if
    it's missing; if it exists without the section, append it; if the section is
    already there, leave it alone.
  - **`.gitignore` semantics** — `.ai/skills/` must stay committed even where
    other `.ai/` content is local. If the repo ignores `.ai/` wholesale, rewrite
    that entry to the exception-safe pair:

    ```
    .ai/*
    !.ai/skills/
    ```

    (A bare `!.ai/skills/` under an ignored `.ai/` has no effect — git never
    descends into ignored directories.) Path-specific ignores like `.ai/specs/`
    need no change.

### 5. Connect Linear (optional)

Offer to connect this repo to Linear for issue tracking. If the user declines,
skip this step entirely — write nothing Linear-related.

If they accept:

1. **Verify the Linear MCP with a live call** — list the workspace's teams via
   the Linear MCP. If the MCP is not configured or the call fails, give the
   user exact setup instructions for adding the Linear MCP server, then
   continue init without Linear — they can re-run `/adlc:init` later to
   retrofit just this step.
2. **Choose the team** — exactly one team returned: use it and say which.
   Multiple teams: ask the user to pick one.
3. **Choose the project** — always ask. List the team's existing projects and
   ask the user to select one **or** create a new one (default the new
   project's name to the repo name). Create it via the MCP if requested.
4. **Write `.adlc/config.json`** in the repo root:

   ```json
   {
     "pm": {
       "provider": "linear",
       "team": "<team key>",
       "teamName": "<team name>",
       "project": "<project id>",
       "projectName": "<project name>"
     }
   }
   ```

   If `.adlc/config.json` already exists with a `pm` block, show it and ask
   before changing it.
5. **Gitignore** — ensure the repo's `.gitignore` contains `.adlc/*` and
   `!.adlc/config.json` (the config is shared; anything else under `.adlc/`
   is per-machine). Add the lines only if missing.

If any MCP call fails mid-flow, report it plainly, write nothing partial, and
finish the rest of init normally. Never write a config containing unverified
IDs.

### 6. Report and hand off

Summarize what you created and tell the user the immediate next moves:

- Review `CLAUDE.md` — especially the Task Router rows and Validation Commands.
- The harness ships more skills: the lifecycle (`/adlc:brainstorm` →
  `/adlc:archive`, Linear required), `/adlc:pr`, `/adlc:add-lesson`. Mention
  they're available.
- Suggest committing the scaffold.

## Boundaries

- **Never** overwrite an existing root `CLAUDE.md` without explicit confirmation.
- **Never** move or delete an existing `.claude/skills/` directory (or its
  contents) without explicit confirmation — the retrofit to a symlink is always
  offered, never assumed.
- **Ask First** before inventing validation commands you could not find — a wrong
  build command is worse than an empty one.
- **Always** fill placeholders from real detection; leave a `TODO(adlc)` marker
  where you genuinely could not determine a value, so it's greppable.
- **Never** write `.adlc/config.json` with team or project IDs that did not
  come from a live Linear MCP response.
- **Ask First** before replacing an existing `pm` block in `.adlc/config.json`.

## Done when

`CLAUDE.md`, `.claude/lessons.md`, and a `.worktrees/` gitignore
entry exist, the Task Router and Validation Commands reflect this specific
project, and you've told the user what to review. The project-skills convention
is in place: `.ai/skills/README.md` exists, `.claude/skills` is a relative
symlink to `../.ai/skills` (or the Windows fallback was reported), `AGENTS.md`
carries the pointer section, and `.ai/skills/` is not gitignored. If the user opted into Linear:
`.adlc/config.json` exists with the `pm` block, every ID in it came from a live
Linear MCP response, and `.gitignore` covers `.adlc/` (except `config.json`).
