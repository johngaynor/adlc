# Conductor compatibility

**Date**: 2026-08-03
**Status**: approved design, not yet implemented

## What

Make ADLC a first-class harness inside [Conductor](https://conductor.build) — the
Mac app that runs multiple Claude Code agents in parallel, one git worktree +
branch per workspace. Four changes:

1. An optional **Conductor setup step in `/adlc:init`** that scaffolds
   `.conductor/settings.toml` (setup/run scripts) from the stack detection init
   already performs.
2. **Worktree awareness in `/adlc:ship-pr`** so it never fights Conductor's
   branch-per-workspace model.
3. A **one-file-per-lesson refactor** of the lessons loop so parallel workspaces
   can add lessons without merge conflicts.
4. A short **"Using with Conductor" README section**.

## Why

ADLC is already mostly compatible: Conductor runs real Claude Code, plugins are
user-scoped so `/adlc:*` skills load in every workspace, and everything init
scaffolds is committed, so it travels into every worktree. What's missing:

- **Fresh worktrees are bare.** Gitignored local files (`.env`,
  `.claude/settings.local.json`) and installed dependencies don't carry over.
  Conductor's answer is a `.conductor/settings.toml` with lifecycle scripts —
  and init already knows the package manager and the real commands, so it is
  perfectly positioned to write one.
- **`ship-pr` assumes it owns branching.** In a Conductor workspace the branch
  already exists and is managed by Conductor; creating or renaming branches
  there is wrong.
- **`lessons.md` is a conflict magnet.** Newest-at-top appends from two parallel
  workspaces conflict on the same line every time.

## Design

### 1. Conductor step in `/adlc:init`

A new optional step in `skills/init/SKILL.md`, inserted between the current
step 4 ("Write the files") and the report step — the same placement and
offer-then-skip shape as the Linear step
(`.ai/specs/2026-08-03-linear-init-integration.md`). Both specs insert into the
same slot; when both are implemented the Linear step comes first (it shapes the
workflow, Conductor shapes the environment). Numbering shifts accordingly.

**Flow:**

1. **Offer** — ask whether to set up Conductor lifecycle scripts. If
   `.conductor/settings.toml` or a legacy `conductor.json` already exists, do
   not overwrite: show what would change and offer to merge instead (same
   idempotency rule as `CLAUDE.md`). If declined, skip entirely.
2. **Write `.conductor/settings.toml`** using only what step 1 detected:
   - **setup**: install dependencies with the detected package manager, then
     copy gitignored local config that actually exists in the root checkout —
     e.g. `.env`, `.claude/settings.local.json` — from
     `$CONDUCTOR_ROOT_PATH` into `$CONDUCTOR_WORKSPACE_PATH`. Only reference
     files that were found; only emit an install command that was verified
     against the lockfile.
   - **run**: the detected dev/start command; fall back to the test watcher if
     the project has no server. Omit `run` entirely if nothing was detected —
     same boundary as Validation Commands: **never invent commands**.
   - **archive**: omitted unless the stack demonstrably creates external
     resources needing cleanup (YAGNI).
3. The file is committed with the rest of the scaffold so every teammate's
   Conductor picks it up.

### 2. Worktree awareness in `/adlc:ship-pr`

Amend workflow step 2 ("Branch if needed"). Before branching, detect a managed
worktree:

- `CONDUCTOR_WORKSPACE_NAME` is set in the environment, **or**
- `git rev-parse --git-common-dir` resolves outside the current checkout's
  `.git` (any linked worktree, Conductor or otherwise).

In that case the current branch **is** the feature branch: use it as-is. Do not
create a second branch and do not rename it to match the `feat/` convention —
the naming convention applies only to branches ship-pr itself creates. The
existing rule (branch only when on `main`/`master`) still covers the
plain-checkout case.

### 3. One file per lesson

Replace the single append-file with a directory:

- **`.claude/lessons/`** — each lesson is `<kebab-title>.md` (numeric suffix on
  slug collision), using the existing entry format: title heading, then
  **Context** / **Problem** / **Rule** / **Applies to**.
- **`.claude/lessons/README.md`** — replaces `templates/lessons.md.template`
  (template renamed to `lessons-readme.md.template`): documents the entry
  format, carries the "review at session start" instruction, and notes that a
  legacy `.claude/lessons.md` keeps working and can be split by hand.
- **`templates/CLAUDE.md.template`** — the Self-Improvement section points at
  the directory: read every file in `.claude/lessons/` at session start;
  `add-lesson` writes a new file there.
- **`skills/add-lesson/SKILL.md`** — writes a new file per lesson instead of
  appending; slug from the title.
- **`skills/init/SKILL.md`** — creates `.claude/lessons/` + README instead of
  `lessons.md`; if a legacy `lessons.md` exists, leave it in place and only add
  the directory.

No automated migration: the plugin is v0.1 with no installed base.

### 4. README

A short "Using with Conductor" section: what works out of the box (plugin
skills, committed scaffold), what the init step adds (lifecycle scripts), and
that one-workspace-per-task is the intended way to run ADLC work streams in
parallel.

## Out of scope

- **Linear lifecycle × Conductor**: with workspace-per-task, in-flight lifecycle
  state must key to the branch/workspace rather than "the repo checkout." That
  is a consideration for `docs/design/2026-08-03-linear-lifecycle-design.md`,
  not this spec.
- Conductor-specific features beyond scripts (checks, agent modes, ports).

## Validation

These are markdown skills, so validation is dogfooding: run `/adlc:init` in a
throwaway repo, accept the Conductor step, open the repo in Conductor, create a
workspace, and confirm (a) setup and run scripts execute, (b) `/adlc:ship-pr`
ships from the workspace without creating a second branch, (c) two parallel
workspaces can each `/adlc:add-lesson` and both branches merge cleanly.
