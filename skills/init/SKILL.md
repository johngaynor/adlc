---
name: init
description: Use when a repo has no ADLC harness yet (no root CLAUDE.md, or the user asks to "set up ADLC", "initialize the harness", "add the AI conventions"). Scaffolds an opinionated CLAUDE.md with a Task Router and boundary-labeled rules, a lessons directory, and a specs directory — tailored to the project's detected stack.
---

# Initialize the ADLC harness

Scaffold this project with the ADLC engineering harness: an opinionated root
`CLAUDE.md` (Task Router + Always/Ask-First/Never/Validation), a `.claude/lessons/`
self-improvement log (one file per lesson), and a `.ai/specs/` directory. Read
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
  Also note the dev/start command if one exists; the Conductor step's
  `{{RUN_COMMAND}}` consumes it.
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
- If `.claude/lessons/` or `.ai/specs/` already exist, leave them and only add
  what's missing. A legacy `.claude/lessons.md` is left in place untouched — the
  lessons README explains it stays readable.

### 4. Write the files

From the plugin's templates (available at `${CLAUDE_PLUGIN_ROOT}/templates/` — the
plugin's install directory; reproduce the structure inline if that path is not
resolvable at runtime), render and write:

- **`CLAUDE.md`** (repo root) — from `templates/CLAUDE.md.template`, with every
  `{{PLACEHOLDER}}` filled from step 1. The Task Router must have real rows for
  this project (e.g. "Add an API route → …", "Add a UI screen → …", "Write a
  migration → …"), not the generic examples. Seed 4–8 rows that match the stack.
- **`.claude/lessons/README.md`** — from `templates/lessons-readme.md.template`
  (create the `.claude/lessons/` directory; the README documents the
  one-file-per-lesson format and the session-start review rule).
- **`.ai/specs/README.md`** — from `templates/spec.md.template`'s sibling note, or
  a short pointer explaining the `{YYYY-MM-DD}-{kebab-title}.md` convention. Create
  the `.ai/specs/` directory.

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

### 6. Offer Conductor lifecycle scripts (optional)

[Conductor](https://conductor.build) runs Claude Code in one git worktree per
workspace; fresh worktrees start without gitignored files or installed
dependencies. This step scaffolds the `.conductor/settings.toml` that fixes that.
It is **optional — offer it and skip cleanly if declined**. (If the repo also
uses the Linear step, that step runs first: Linear shapes the workflow,
Conductor shapes the environment.)

1. **Offer** — ask whether to set up Conductor workspace scripts. If
   `.conductor/settings.toml` or a legacy `conductor.json` already exists, do
   **not** overwrite: show what you'd change and offer to merge instead (same
   rule as `CLAUDE.md`). If declined, skip — write nothing Conductor-related.
2. **Write `.conductor/settings.toml`** using only what step 1 detected:

   ```toml
   [scripts]
   # Runs once when a Conductor workspace (git worktree) is created.
   setup = '''
   [ -f "$CONDUCTOR_ROOT_PATH/.env" ] && cp "$CONDUCTOR_ROOT_PATH/.env" "$CONDUCTOR_WORKSPACE_PATH/.env" || true
   mkdir -p "$CONDUCTOR_WORKSPACE_PATH/.claude"
   [ -f "$CONDUCTOR_ROOT_PATH/.claude/settings.local.json" ] && cp "$CONDUCTOR_ROOT_PATH/.claude/settings.local.json" "$CONDUCTOR_WORKSPACE_PATH/.claude/settings.local.json" || true
   {{INSTALL_COMMAND}}
   '''
   # Runs on the workspace Run button.
   run = '{{RUN_COMMAND}}'
   ```

   - `{{INSTALL_COMMAND}}` — the detected package manager's install (e.g.
     `pnpm install`, `uv sync`). If the project has no dependency install,
     drop the line. It goes last in `setup`, after the copy guards.
   - `{{RUN_COMMAND}}` — the detected dev/start command; fall back to the test
     watcher if there is no server; **omit the `run` key entirely** if nothing
     was detected. Never invent commands (same boundary as Validation
     Commands).
   - Keep only the copy lines for local files that plausibly exist in this
     repo (e.g. drop the `.env` line if the project doesn't use one). Copied
     files are gitignored, so they never leak into the worktree's branch.
   - Every guard line (`[ -f X ] && cp X Y`) must end in `|| true` so it stays
     status-neutral — the script's exit code reflects a fine run, not the
     presence or absence of an optional file. Use the `'''…'''` literal
     string for `setup` and `'…'` for `run` so backslashes in detected
     commands pass through unescaped.
   - No `archive` script unless the stack demonstrably creates external
     resources needing cleanup.
3. **Commit with the scaffold** — the file is committed so every teammate's
   Conductor picks it up.

### 7. Report and hand off

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
- **Never** write `.adlc/config.json` with team or project IDs that did not
  come from a live Linear MCP response.
- **Ask First** before replacing an existing `pm` block in `.adlc/config.json`.
- **Never** write Conductor scripts the user declined, and never put commands in
  `settings.toml` that detection did not find.

## Done when

`CLAUDE.md`, `.claude/lessons/`, and `.ai/specs/` exist, the Task Router and
Validation Commands reflect this specific project, and you've told the user what to
review. If the user opted into Linear: `.adlc/config.json` exists with the `pm`
block, every ID in it came from a live Linear MCP response, and `.gitignore`
covers `.adlc/` (except `config.json`). If the Conductor step was accepted,
`.conductor/settings.toml` exists with only detected commands.
