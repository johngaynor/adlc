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
- **Project skills scaffold** — TBD(project-skills): the convention for a repo's
  own agent skills (canonical home, discovery wiring, docs pointer) is parked
  while it is redesigned. Init scaffolds nothing for it — do not create skill
  directories, symlinks, or pointer sections.

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

### 6. Scaffold permissions

Set up permission settings so the ADLC lifecycle runs without constant
prompts. Two layers, two files — every write is **additive-if-missing**:
add only rules that aren't already present, never remove or rewrite existing
ones, so a re-run of `/adlc:init` produces zero diff.

1. **Committed `.claude/settings.json` — the safe git/gh allowlist.** Ask
   first (this file changes the team's permission posture for everyone;
   default to yes):
   - If the file is absent, write it from `templates/settings.json.template`
     verbatim — `allow` for `Bash(git *)` / `Bash(gh *)`, `ask` guards for
     `gh repo delete *`, `gh secret *`, and the `gh api` DELETE forms.
   - If it exists, read-merge-write: add only the template's missing
     `allow`/`ask` entries, preserve everything already there, and show the
     diff before saving. If the file is not valid JSON, report that plainly
     and write nothing.
2. **Per-user `.claude/settings.local.json` — the MCP grants.** Detect which
   MCP servers are actually configured (`claude mcp list` — one
   `name: target - status` line per server) and scaffold grants only for
   servers that exist, using the detected name as the rule prefix (a
   claude.ai connector named `X` appears as `mcp__claude_ai_X__<tool>`).
   Merge into the file with the same additive-if-missing rule, creating it
   if absent:
   - **linear** (required by the PM seam) — allow exactly the seam's tool
     set: `get_issue`, `save_issue`, `list_issues`, `list_issue_labels`,
     `create_issue_label`, `list_issue_statuses`, `list_teams`, `get_team`,
     `list_projects`, `save_project`, `list_milestones`, `get_milestone`,
     `list_comments`, `save_comment` — each as
     `mcp__<server>__<tool>`. Never grant `delete_*` tools; those keep
     prompting. If no Linear server is configured, report the setup
     instructions and skip — same posture as step 5.
   - **github** — read-only grants: `mcp__<server>__get_*`,
     `mcp__<server>__list_*`, `mcp__<server>__search_*`,
     `mcp__<server>__pull_request_read`, `mcp__<server>__issue_read`
     (allow rules accept tool-name globs after a literal server prefix).
     Writes stay on the `gh` CLI, which layer 1 already covers.
   - **notion** — read-only grants: the server's search and fetch tools
     (e.g. `notion-search`, `notion-fetch`). Notion writes keep prompting.
   - Servers not configured are skipped silently; servers beyond these
     three are out of scope — the user adds their own rules.
3. **Gitignore** — `settings.local.json` is not ignored automatically:
   ensure the repo's `.gitignore` contains `.claude/settings.local.json`,
   adding the line only if missing.

### 7. Report and hand off

Summarize what you created and tell the user the immediate next moves:

- Review `CLAUDE.md` — especially the Task Router rows and Validation Commands.
- Which permission files were written or merged (step 6), and that MCP
  grants live in `settings.local.json` — a user who wants them to follow
  their account across repos can promote them to `~/.claude/settings.json`
  themselves; init never writes outside the repo.
- The harness ships more skills: the lifecycle (`/adlc:brainstorm` →
  `/adlc:pr`, Linear required for all but `/adlc:pr`), `/adlc:add-lesson`.
  Mention they're available.
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
- **Ask First** before writing the committed `.claude/settings.json` — it sets
  the whole team's permission posture.
- **Never** write permission rules outside the repo (`~/.claude/` stays the
  user's own), never grant destructive MCP tools (`delete_*` and kin), and
  never remove or rewrite a permission rule the user already has — permission
  writes are additive-if-missing only.

## Done when

`CLAUDE.md`, `.claude/lessons.md`, and a `.worktrees/` gitignore
entry exist, the Task Router and Validation Commands reflect this specific
project, and you've told the user what to review. (Project-skills scaffolding
is parked — TBD(project-skills) — and intentionally absent.) Permissions
are scaffolded: `.claude/settings.json` carries the git/gh allowlist (or the
user declined), `settings.local.json` carries grants for the MCP servers that
were actually detected, and `.gitignore` covers `.claude/settings.local.json`.
If the user opted into Linear:
`.adlc/config.json` exists with the `pm` block, every ID in it came from a live
Linear MCP response, and `.gitignore` covers `.adlc/` (except `config.json`).
