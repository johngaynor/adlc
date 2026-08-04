# Conductor Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make ADLC run cleanly inside Conductor (conductor.build) — workspace lifecycle scripts scaffolded by init, a worktree-aware ship-pr, and a merge-conflict-free lessons format.

**Architecture:** ADLC is a docs-only Claude Code plugin: every deliverable is a markdown skill, template, or doc. The lessons loop moves from one append-file (`.claude/lessons.md`) to one-file-per-lesson (`.claude/lessons/`), `/adlc:init` gains an optional Conductor step that writes `.conductor/settings.toml`, and `/adlc:ship-pr` learns to detect managed worktrees.

**Tech Stack:** Markdown skills (Claude Code plugin format per `CONVENTIONS.md`), TOML (Conductor settings), git.

**Spec:** `.ai/specs/2026-08-03-conductor-compatibility.md` — read it first.

## Global Constraints

- Work in the worktree `/Users/johngaynor/Dev/adlc/.claude/worktrees/conductor-compatibility` on branch `design/conductor-compatibility`. All paths below are relative to that worktree root.
- There is no build or test suite: **verification = the grep/read step in each task**. Run it and confirm the expected output before committing.
- Skills must follow `CONVENTIONS.md` (frontmatter `name` + `description` with trigger phrases; Workflow / Boundaries / Done-when sections; link to `METHODOLOGY.md` for vocabulary, never redefine it).
- Lesson entry format everywhere is exactly: **Context / Problem / Rule / Applies to** (that order, those labels).
- Conductor TOML shape (verified against `https://conductor.build/schemas/settings.repo.schema.json`): `[scripts]` table with `setup` (string), `run` (string or named-table form), `archive` (string). Env vars available to scripts: `CONDUCTOR_ROOT_PATH`, `CONDUCTOR_WORKSPACE_PATH`, `CONDUCTOR_WORKSPACE_NAME`, `CONDUCTOR_PORT`.
- Never invent commands in scaffolded output — a command appears in `settings.toml` only if init's detection found it (same boundary as Validation Commands).
- Commit after every task, conventional-commits style, with the trailer used in this repo:
  ```
  Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01N4qhu4GRvktp3wLLmbHD15
  ```

---

### Task 1: Lessons templates — per-lesson README template + CLAUDE.md template

**Files:**
- Delete: `templates/lessons.md.template`
- Create: `templates/lessons-readme.md.template`
- Modify: `templates/CLAUDE.md.template` (Self-Improvement section, lines 78–83)

**Interfaces:**
- Produces: template filename `lessons-readme.md.template` (Tasks 2, 3, and 6 reference it) and the target path `.claude/lessons/README.md` it renders to.

- [ ] **Step 1: Replace the template file**

```bash
git rm templates/lessons.md.template
```

Create `templates/lessons-readme.md.template` with exactly:

```markdown
# Lessons

Recurring patterns and mistakes to avoid on this codebase — **one file per
lesson** in this directory. **Review every file here at session start.**

Add a new file whenever the agent is corrected on something that will recur.
One lesson per file means parallel branches (e.g. Conductor workspaces) never
merge-conflict on a shared log.

- **Filename**: `<kebab-title>.md`, short and specific
  (e.g. `no-raw-sql-in-handlers.md`). On a name collision, append `-2`, `-3`, ….
- **Format** — every lesson file follows this exactly:

  ```
  # <short, specific title>
  **Context**: where/when this came up.
  **Problem**: what went wrong.
  **Rule**: the durable rule that prevents it next time.
  **Applies to**: the paths or areas this governs.
  ```

A legacy single-file `.claude/lessons.md` keeps working — if present, read it
too at session start, and split it into per-file entries here whenever
convenient.
```

- [ ] **Step 2: Update the Self-Improvement section of `templates/CLAUDE.md.template`**

Replace the whole section (currently the last 6 lines of the file):

```markdown
## Self-Improvement (the lessons loop)

Recurring mistakes get written down, not repeated. When corrected on something that
will come up again, add a new file to [`.claude/lessons/`](.claude/lessons/) — one
lesson per file (Context / Problem / Rule / Applies-to; format documented in that
directory's README). Review every file there at the start of a session.
The `/adlc:add-lesson` skill drives this.
```

- [ ] **Step 3: Verify**

Run: `grep -rn "lessons.md" templates/`
Expected: no matches. Run: `ls templates/`
Expected: `CLAUDE.md.template  lessons-readme.md.template  spec.md.template`

- [ ] **Step 4: Commit**

```bash
git add -A templates/
git commit -m "feat: one-file-per-lesson templates (.claude/lessons/ directory)"
```

---

### Task 2: Rewrite `/adlc:add-lesson` for per-file lessons

**Files:**
- Modify: `skills/add-lesson/SKILL.md` (full rewrite of body; frontmatter description updated)

**Interfaces:**
- Consumes: `templates/lessons-readme.md.template` from Task 1.
- Produces: the behavior "one new file per lesson under `.claude/lessons/`" that Task 4's METHODOLOGY wording and Task 6's README describe.

- [ ] **Step 1: Rewrite `skills/add-lesson/SKILL.md`**

Replace the file's full contents with:

```markdown
---
name: add-lesson
description: Use when the user corrects the agent on something likely to recur, or says "remember this", "add a lesson", "don't do that again". Writes a structured entry as a new file in .claude/lessons/ so the mistake isn't repeated.
---

# Add a lesson

Close the self-improvement loop: capture a correction as a durable, structured
lesson — one file per lesson in `.claude/lessons/`. See
[`METHODOLOGY.md`](../../METHODOLOGY.md) § Lessons loop.

## Arguments

- `$ARGUMENTS` (optional) — an explicit lesson to record. If absent, infer it from
  the correction in the conversation.

## Workflow

1. **Identify the lesson.** From `$ARGUMENTS` or the recent correction, extract the
   *generalizable* rule — the class of mistake, not the one-off instance.
2. **Sanity-check it's worth recording.** Skip if it's ephemeral conversational
   context or a one-time detail. Record only durable rules that will apply again. If
   it's unclear whether it generalizes, ask.
3. **Check for duplicates.** Read every file in `.claude/lessons/` (and a legacy
   `.claude/lessons.md` if present). If an existing entry covers the same ground,
   **update** that file rather than adding a near-duplicate.
4. **Write the lesson file** at `.claude/lessons/<kebab-title>.md` — filename from
   the title, short and specific; on a collision append `-2`, `-3`, …:
   ```
   # <short, specific title>
   **Context**: where/when this came up.
   **Problem**: what went wrong.
   **Rule**: the durable rule that prevents it next time.
   **Applies to**: the paths or areas this governs.
   ```
   If `.claude/lessons/` doesn't exist, create it and add its `README.md` from
   `${CLAUDE_PLUGIN_ROOT}/templates/lessons-readme.md.template` (or reproduce the
   format doc inline if that path is not resolvable).
5. **Confirm** what was recorded and where.

## Boundaries

- **Never** record secrets, credentials, or one-off conversational context.
- **Never** append to a legacy `.claude/lessons.md` — new lessons always get their
  own file (updates to an existing legacy entry may edit it in place).
- **Ask First** only when it's genuinely unclear whether the correction generalizes.

## Done when

A well-formed, non-duplicate lesson file exists under `.claude/lessons/` and the
user has seen it.
```

- [ ] **Step 2: Verify**

Run: `grep -c "lessons/" skills/add-lesson/SKILL.md && grep -n "Append" skills/add-lesson/SKILL.md`
Expected: several `lessons/` matches; no "Append to .claude/lessons.md" workflow step remains.

- [ ] **Step 3: Commit**

```bash
git add skills/add-lesson/SKILL.md
git commit -m "feat: add-lesson writes one file per lesson"
```

---

### Task 3: `/adlc:init` — scaffold the lessons directory

**Files:**
- Modify: `skills/init/SKILL.md` (frontmatter description; intro paragraph; steps 3 & 4; Done-when)

**Interfaces:**
- Consumes: `templates/lessons-readme.md.template` from Task 1.
- Produces: init scaffolds `.claude/lessons/README.md` (Task 5 inserts its Conductor step around this same file's numbering — do this task first).

- [ ] **Step 1: Update frontmatter description**

In `skills/init/SKILL.md` line 3, change `a lessons.md, and a specs directory` to `a lessons directory, and a specs directory`.

- [ ] **Step 2: Update intro paragraph**

Change `` a `.claude/lessons.md` self-improvement log `` to `` a `.claude/lessons/` self-improvement log (one file per lesson) ``.

- [ ] **Step 3: Update step 3 (idempotency)**

Replace the bullet `` If `.claude/lessons.md` or `.ai/specs/` already exist, leave them and only add what's missing. `` with:

```markdown
- If `.claude/lessons/` or `.ai/specs/` already exist, leave them and only add
  what's missing. A legacy `.claude/lessons.md` is left in place untouched — the
  lessons README explains it stays readable.
```

- [ ] **Step 4: Update step 4 (write the files)**

Replace the `.claude/lessons.md` bullet with:

```markdown
- **`.claude/lessons/README.md`** — from `templates/lessons-readme.md.template`
  (create the `.claude/lessons/` directory; the README documents the
  one-file-per-lesson format and the session-start review rule).
```

- [ ] **Step 5: Update Done-when**

Change `` `CLAUDE.md`, `.claude/lessons.md`, and `.ai/specs/` exist `` to `` `CLAUDE.md`, `.claude/lessons/`, and `.ai/specs/` exist ``.

- [ ] **Step 6: Verify**

Run: `grep -n "lessons.md" skills/init/SKILL.md`
Expected: the only match is the legacy-file mention in step 3 (the idempotency bullet).

- [ ] **Step 7: Commit**

```bash
git add skills/init/SKILL.md
git commit -m "feat: init scaffolds .claude/lessons/ directory"
```

---

### Task 4: METHODOLOGY.md — lessons loop wording

**Files:**
- Modify: `METHODOLOGY.md:75-88` (the "lessons loop" half of § 4)

**Interfaces:**
- Consumes: the per-file format from Task 1 (heading is `#`, not `##`).

- [ ] **Step 1: Update the lessons-loop paragraphs**

Replace from `**The lessons loop.**` through `It is reviewed at session start.` with:

```markdown
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
```

- [ ] **Step 2: Verify**

Run: `grep -n "lessons.md" METHODOLOGY.md`
Expected: no matches.

- [ ] **Step 3: Commit**

```bash
git add METHODOLOGY.md
git commit -m "docs: methodology reflects one-file-per-lesson format"
```

---

### Task 5: `/adlc:init` — optional Conductor step

**Files:**
- Modify: `skills/init/SKILL.md` (insert new step 5 between "Write the files" and "Report and hand off"; renumber the report step to 6; extend Boundaries and Done-when)

**Interfaces:**
- Consumes: init's step-1 detection outputs (package manager, install command, dev/start command) — already defined in the skill.
- Produces: scaffolded `.conductor/settings.toml`; README Task 6 describes this step to users.

- [ ] **Step 1: Insert the Conductor step**

After the end of step 4 ("Write the files") and before "### 5. Report and hand off", insert (and rename the report heading to `### 6. Report and hand off`):

```markdown
### 5. Offer Conductor lifecycle scripts (optional)

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
   setup = """
   [ -f "$CONDUCTOR_ROOT_PATH/.env" ] && cp "$CONDUCTOR_ROOT_PATH/.env" "$CONDUCTOR_WORKSPACE_PATH/.env"
   mkdir -p "$CONDUCTOR_WORKSPACE_PATH/.claude"
   [ -f "$CONDUCTOR_ROOT_PATH/.claude/settings.local.json" ] && cp "$CONDUCTOR_ROOT_PATH/.claude/settings.local.json" "$CONDUCTOR_WORKSPACE_PATH/.claude/settings.local.json"
   {{INSTALL_COMMAND}}
   """
   # Runs on the workspace Run button.
   run = "{{RUN_COMMAND}}"
   ```

   - `{{INSTALL_COMMAND}}` — the detected package manager's install (e.g.
     `pnpm install`, `uv sync`). If the project has no dependency install,
     drop the line.
   - `{{RUN_COMMAND}}` — the detected dev/start command; fall back to the test
     watcher if there is no server; **omit the `run` key entirely** if nothing
     was detected. Never invent commands (same boundary as Validation
     Commands).
   - Keep only the copy lines for local files that plausibly exist in this
     repo (e.g. drop the `.env` line if the project doesn't use one). Copied
     files are gitignored, so they never leak into the worktree's branch.
   - No `archive` script unless the stack demonstrably creates external
     resources needing cleanup.
3. **Commit with the scaffold** — the file is committed so every teammate's
   Conductor picks it up.
```

- [ ] **Step 2: Extend Boundaries and Done-when**

Add to the skill's Boundaries section:

```markdown
- **Never** write Conductor scripts the user declined, and never put commands in
  `settings.toml` that detection did not find.
```

In Done-when, after the existing sentence about Task Router/Validation Commands, add: `If the Conductor step was accepted, .conductor/settings.toml exists with only detected commands.`

- [ ] **Step 3: Verify**

Run: `grep -n "^### " skills/init/SKILL.md`
Expected: steps numbered 1–6 in order, with `### 5. Offer Conductor lifecycle scripts (optional)` and `### 6. Report and hand off`.

- [ ] **Step 4: Commit**

```bash
git add skills/init/SKILL.md
git commit -m "feat: optional Conductor settings.toml step in init"
```

---

### Task 6: `/adlc:ship-pr` — worktree awareness

**Files:**
- Modify: `skills/ship-pr/SKILL.md:18-19` (workflow step 2)

**Interfaces:**
- Consumes: nothing from other tasks (independent).

- [ ] **Step 1: Replace workflow step 2**

Replace:

```markdown
2. **Branch if needed.** If on the default branch (`main`/`master`), create a
   feature branch first, kebab-case and prefixed by change type: `feat/`, `fix/`,
   `chore/`, `docs/`, `refactor/`.
```

with:

```markdown
2. **Branch if needed.** First detect a managed worktree: `CONDUCTOR_WORKSPACE_NAME`
   is set in the environment, or `git rev-parse --git-common-dir` resolves outside
   this checkout's own `.git`. In a managed worktree the current branch *is* the
   feature branch — use it as-is; do **not** create a second branch or rename it.
   Otherwise, if on the default branch (`main`/`master`), create a feature branch
   first, kebab-case and prefixed by change type: `feat/`, `fix/`, `chore/`,
   `docs/`, `refactor/`. The naming convention applies only to branches this skill
   creates.
```

- [ ] **Step 2: Verify**

Run: `grep -n "CONDUCTOR_WORKSPACE_NAME\|git-common-dir" skills/ship-pr/SKILL.md`
Expected: both appear in step 2.

- [ ] **Step 3: Commit**

```bash
git add skills/ship-pr/SKILL.md
git commit -m "feat: ship-pr respects managed worktree branches"
```

---

### Task 7: README — lessons path, template name, "Using with Conductor"

**Files:**
- Modify: `README.md` (line 26 install blurb; line 63 repo layout; new section after "Skills")

**Interfaces:**
- Consumes: behaviors from Tasks 1–6 (describes them to users).

- [ ] **Step 1: Update existing references**

- Line 26: `a `.claude/lessons.md`, and a `.ai/specs/` directory` → `` a `.claude/lessons/` directory, and a `.ai/specs/` directory ``.
- Repo layout: `│   ├── lessons.md.template` → `│   ├── lessons-readme.md.template`.

- [ ] **Step 2: Add the Conductor section**

Insert after the Skills table (before "## Updating"):

```markdown
## Using with Conductor

ADLC works inside [Conductor](https://conductor.build) out of the box: plugins
are user-scoped, so `/adlc:*` skills load in every workspace, and everything
init scaffolds is committed, so it travels into every worktree.

`/adlc:init` offers one extra step for Conductor users: it scaffolds a
`.conductor/settings.toml` whose setup script installs dependencies and copies
gitignored local files (`.env`, `.claude/settings.local.json`) into new
workspaces, and whose run script starts the detected dev command.

One workspace per task is the intended shape: `/adlc:write-spec` at the start,
`/adlc:ship-pr` from the workspace's own branch at the end, and lessons are one
file each (`.claude/lessons/`), so parallel workspaces never merge-conflict on
the lessons log.
```

- [ ] **Step 3: Verify**

Run: `grep -n "lessons.md\|Conductor" README.md`
Expected: no `lessons.md` matches (except none); Conductor section present.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: README covers Conductor usage and lessons directory"
```

---

### Task 8: Consistency sweep

**Files:**
- Possibly modify: any file a stale reference survives in.

- [ ] **Step 1: Sweep for stale references**

Run: `grep -rn "lessons.md" --include="*.md" --include="*.template" . | grep -v ".ai/specs/" | grep -v docs/plans | grep -v "legacy"`
Expected: no matches (spec/plan/legacy mentions are allowed). Fix any stragglers.

Run: `grep -rn "lessons.md.template" .`
Expected: no matches anywhere.

- [ ] **Step 2: Read-through**

Read `skills/init/SKILL.md` top to bottom once: step numbering 1–6 coherent, Conductor step references match the templates that exist.

- [ ] **Step 3: Commit (only if fixes were needed)**

```bash
git add -A
git commit -m "chore: sweep stale lessons.md references"
```
