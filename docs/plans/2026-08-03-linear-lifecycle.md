# ADLC Linear Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the six-stage ADLC task lifecycle (`brainstorm → spec → plan → execute → pr → archive`) as native Claude Code plugin skills, using Linear as the live workspace and the repo as a curated after-the-fact record.

**Architecture:** Each stage is one human-invoked skill under `skills/`. All durable state lives in a Linear issue whose description holds canonical `##` sections. Skills touch Linear only through a thin, documented operation set (the "PM seam") defined once in a reference doc, so a second PM system can be added later without editing skills. Task identity is per-agent and derived (Linear issue ref in session context before code; git branch in a per-task worktree from `execute` onward) — never shared mutable state.

**Tech Stack:** Claude Code plugin (skills = `SKILL.md` with YAML frontmatter), Linear MCP server (or Linear API) reached via ToolSearch/MCP, git worktrees, `gh` CLI, JSON config.

## Global Constraints

Every task's requirements implicitly include these (copied verbatim from the design + plugin conventions):

- **Self-contained:** ADLC depends on no other Claude Code plugin. Do not reference or call superpowers. The brainstorm stage is a native ADLC skill.
- **Skill conventions:** Follow `CONVENTIONS.md` — kebab-case folder = `/adlc:<name>`; frontmatter has `name` (matches folder) + `description` starting with "Use when…"; body sections in order (purpose, arguments, workflow, boundaries, done when); imperative voice; reference `METHODOLOGY.md` vocabulary, don't redefine it.
- **Namespace:** all skills invoke as `/adlc:<name>`; never self-prefix.
- **Plugin file access:** reference bundled files via `${CLAUDE_PLUGIN_ROOT}/...`, with an inline fallback if unresolvable.
- **Parallel-safe:** no shared mutable pointer for "current task." Pre-code identity = Linear issue ref in session context; code-stage identity = git branch in a per-task worktree. Any cached ref is worktree-local and gitignored.
- **Canonical Linear sections (exact names):** `## Idea`, `## Specification`, `## Plan`, `## Progress`, `## Outcome`.
- **Status values (exact):** `Backlog → Todo → In Progress → In Review → Done`.
- **Branch format:** `<initials>/<issue-identifier>-<slug>` (e.g. `jg/eng-142-linear-lifecycle`) so Linear auto-links the PR.
- **Idempotency:** every Linear write is an upsert of a named `##` section; re-running a stage overwrites cleanly, never duplicates.
- **Never** commit secrets; **never** write user-facing state to a shared repo path during parallel work.

## File Structure

- Create: `reference/pm-seam.md` — the PM operation contract, Linear mapping, section schema, status values, and task-identity resolution. The shared contract every lifecycle skill consumes.
- Create: `skills/brainstorm/SKILL.md` — sharpen idea, create issue.
- Create: `skills/spec/SKILL.md` — write `## Specification`.
- Create: `skills/plan/SKILL.md` — write `## Plan` + `## Progress` checklist.
- Create: `skills/execute/SKILL.md` — branch/worktree, do work, tick progress.
- Rename + Modify: `skills/ship-pr/` → `skills/pr/SKILL.md` — validate/push/open PR + Linear `In Review`, degrade gracefully with no Linear.
- Create: `skills/archive/SKILL.md` — curated summary → repo, close issue.
- Create: `templates/archive-summary.md.template` — the repo summary format.
- Modify: `skills/init/SKILL.md` — add PM config write + Linear MCP detection.
- Modify: `README.md` — document the lifecycle and updated skill table.
- Modify: `skills/write-spec/SKILL.md` — mark superseded by `/adlc:spec` for Linear repos; keep as the no-PM fallback.

---

### Task 1: PM seam + data model + identity reference

**Files:**
- Create: `reference/pm-seam.md`

**Interfaces:**
- Consumes: nothing (foundation).
- Produces: the operation vocabulary every later skill cites by name:
  - `createTask(title, ideaMarkdown) → taskRef` — new Linear issue, status `Backlog`, description seeded with `## Idea`.
  - `writeSection(taskRef, sectionName, markdown)` — idempotent upsert of a single `## <sectionName>` block in the issue description.
  - `readTask(taskRef) → { sections: {name→markdown}, checklist: [{text, done}], status }`.
  - `tickPhase(taskRef, phaseText, done=true)` — set one `## Progress` checkbox.
  - `setStatus(taskRef, status)` — one of the exact status values.
  - `closeTask(taskRef)` — set status `Done`.
  - `findTasks(filter) → taskRef[]` — query (e.g. has `## Specification`, no `## Plan`) for resume.
  - `resolveCurrentTask() → taskRef` — pre-code: the ref held in session context; code stages: parse the git branch (`<initials>/<identifier>-<slug>`) → identifier → issue.

- [ ] **Step 1: Write the reference doc**

Create `reference/pm-seam.md` containing, in this order:
1. Purpose paragraph: the seam isolates all PM calls so skills never call Linear ad hoc; Linear is the only implementation for v1.
2. **Operation table** — each operation above with signature, description, and the concrete Linear MCP action it maps to (create issue; update issue description; read issue; update description checkbox; update issue state; set state `Done`; search issues; parse branch → issue).
3. **Linear data model** — the issue-as-canonical-document layout with the exact section names and the note that `writeSection` upserts by matching the `## <name>` heading.
4. **Status mapping** — the exact values and which stage sets each.
5. **Task identity resolution** — the pre-code (session ref) vs code (branch parse) split, branch format, and the `findTasks` resume path. State explicitly: no shared pointer file; any cache is worktree-local + gitignored.
6. **Preconditions contract** — a table: each stage → the section(s) that must already exist before it runs.

- [ ] **Step 2: Verify structure**

Run: `grep -nE '^(## |### |createTask|writeSection|readTask|tickPhase|setStatus|closeTask|findTasks|resolveCurrentTask)' reference/pm-seam.md`
Expected: all nine operation names and the section headings appear.

- [ ] **Step 3: Verify the exact contract constants are present**

Run: `grep -c -E '## Idea|## Specification|## Plan|## Progress|## Outcome|Backlog|Todo|In Progress|In Review|Done' reference/pm-seam.md`
Expected: count ≥ 10 (all section names + all status values present).

- [ ] **Step 4: Commit**

```bash
git add reference/pm-seam.md
git commit -m "docs: add PM seam, data model, and task-identity reference"
```

---

### Task 2: `/adlc:brainstorm` skill

**Files:**
- Create: `skills/brainstorm/SKILL.md`

**Interfaces:**
- Consumes: `createTask`, `setStatus` from `reference/pm-seam.md`.
- Produces: a Linear issue with `## Idea` at status `Backlog`; returns its `taskRef` into session context for the next stage.

- [ ] **Step 1: Write the skill**

Frontmatter: `name: brainstorm`; `description: Use when starting a new piece of work from a rough idea — the user wants to kick off an ADLC task. Runs a collaborative dialogue to sharpen the idea, then creates the Linear issue that will hold the whole task.`

Body (per `CONVENTIONS.md`):
- **Purpose:** sharpen a rough idea into a crisp problem statement, then create the task's Linear issue.
- **Workflow:**
  1. Dialogue with the user to clarify purpose, constraints, and success criteria — one question at a time; do not invent scope.
  2. Draft a concise title and an `## Idea` markdown block (problem, why now, rough success criteria).
  3. Confirm the idea with the user (gate) **before** creating the issue.
  4. Call `createTask(title, ideaMarkdown)` (see `reference/pm-seam.md`); status starts `Backlog`.
  5. Report the issue URL and hold its `taskRef` in context for `/adlc:spec`.
- **Boundaries:** Never create the issue before the user approves the idea. Never write code or a branch here. Ask First before broadening scope.
- **Done when:** a `Backlog` issue exists with `## Idea`, and the user has the URL.

- [ ] **Step 2: Validate frontmatter and references**

Run: `head -4 skills/brainstorm/SKILL.md | grep -E '^name: brainstorm$' && grep -c 'reference/pm-seam.md\|createTask' skills/brainstorm/SKILL.md`
Expected: name matches; ≥1 reference to the seam.

- [ ] **Step 3: Commit**

```bash
git add skills/brainstorm/SKILL.md
git commit -m "feat: add /adlc:brainstorm lifecycle skill"
```

---

### Task 3: `/adlc:spec` skill

**Files:**
- Create: `skills/spec/SKILL.md`

**Interfaces:**
- Consumes: `readTask`, `writeSection`. Precondition: issue has `## Idea`, no `## Specification` required yet.
- Produces: `## Specification` on the issue.

- [ ] **Step 1: Write the skill**

Frontmatter: `name: spec`; `description: Use when an ADLC task has an idea and needs its specification written — after /adlc:brainstorm. Researches and writes the spec into the task's Linear issue.`

Body:
- **Purpose:** turn `## Idea` into a reviewed specification on the same issue.
- **Arguments:** optional `taskRef` (issue URL/ID). If absent, use the session's current task, or `findTasks(has Idea, no Specification)` and ask which.
- **Workflow:**
  1. `readTask` — require `## Idea`; refuse if missing (precondition).
  2. Research the affected areas via the consumer repo's Task Router.
  3. Draft the specification: problem/goal, approach (with rejected alternatives + blast radius), and the tests/coverage the work will need.
  4. `writeSection(taskRef, "Specification", md)`.
  5. Present it for review in Linear (gate) before planning.
- **Boundaries:** Ask First before expanding scope beyond `## Idea`. Never start the plan or code here.
- **Done when:** `## Specification` exists and the user has reviewed it.

- [ ] **Step 2: Validate**

Run: `grep -E '^name: spec$' skills/spec/SKILL.md && grep -c 'Specification\|readTask\|writeSection' skills/spec/SKILL.md`
Expected: name matches; section + both ops referenced.

- [ ] **Step 3: Commit**

```bash
git add skills/spec/SKILL.md
git commit -m "feat: add /adlc:spec lifecycle skill"
```

---

### Task 4: `/adlc:plan` skill

**Files:**
- Create: `skills/plan/SKILL.md`

**Interfaces:**
- Consumes: `readTask`, `writeSection`, `setStatus`. Precondition: issue has `## Specification`.
- Produces: `## Plan` + `## Progress` (a `- [ ]` checklist, one box per phase); status `Todo`.

- [ ] **Step 1: Write the skill**

Frontmatter: `name: plan`; `description: Use when an ADLC task has an approved spec and needs its technical plan — after /adlc:spec. Writes a phased plan and a progress checklist into the Linear issue.`

Body:
- **Purpose:** produce the technical plan and the live progress checklist.
- **Arguments:** optional `taskRef`; else current task or `findTasks(has Specification, no Plan)`.
- **Workflow:**
  1. `readTask` — require `## Specification`; refuse otherwise.
  2. Break the work into ordered, independently-committable phases.
  3. `writeSection(taskRef, "Plan", md)` with the phase-by-phase plan.
  4. `writeSection(taskRef, "Progress", checklist)` — one `- [ ]` per phase, text matching the phase names.
  5. `setStatus(taskRef, "Todo")`.
  6. Present for approval (gate).
- **Boundaries:** Never start code here. Ask First before changing the agreed spec.
- **Done when:** `## Plan` and `## Progress` exist, status is `Todo`, user approved.

- [ ] **Step 2: Validate**

Run: `grep -E '^name: plan$' skills/plan/SKILL.md && grep -c 'Plan\|Progress\|setStatus\|Todo' skills/plan/SKILL.md`
Expected: name matches; sections, `setStatus`, and `Todo` referenced.

- [ ] **Step 3: Commit**

```bash
git add skills/plan/SKILL.md
git commit -m "feat: add /adlc:plan lifecycle skill"
```

---

### Task 5: `/adlc:execute` skill

**Files:**
- Create: `skills/execute/SKILL.md`

**Interfaces:**
- Consumes: `readTask`, `tickPhase`, `setStatus`, `resolveCurrentTask`. Precondition: issue has `## Plan` + `## Progress`.
- Produces: completed phases; ticked `## Progress`; status `In Progress`; a git branch + worktree named per the branch format.

- [ ] **Step 1: Write the skill**

Frontmatter: `name: execute`; `description: Use when an ADLC task is planned and it's time to write code — after /adlc:plan. Creates the task branch/worktree and works the plan phase-by-phase, ticking progress live in Linear.`

Body:
- **Purpose:** do the coding work, one phase at a time, with live progress in Linear.
- **Workflow:**
  1. `readTask` — require `## Plan` + `## Progress`; refuse otherwise.
  2. **Create the branch + worktree now** (first code stage): branch `<initials>/<issue-identifier>-<slug>` in its own git worktree. This is what makes parallel execution isolated and links the PR later.
  3. `setStatus(taskRef, "In Progress")`.
  4. For each unchecked phase in `## Progress`: implement it, verify it, then `tickPhase(taskRef, phaseText)`. Checkpoint with the user between phases.
  5. Stop when all phases are checked. Do not open the PR here.
- **Boundaries:** Never run outside a task worktree. Ask First before deviating from the plan; if you must deviate, note it for the archive. Never mark a phase done with failing verification.
- **Done when:** all `## Progress` boxes are ticked and the work is committed on the task branch.

- [ ] **Step 2: Validate**

Run: `grep -E '^name: execute$' skills/execute/SKILL.md && grep -c 'worktree\|tickPhase\|In Progress\|branch' skills/execute/SKILL.md`
Expected: name matches; worktree, tickPhase, status, branch all referenced.

- [ ] **Step 3: Commit**

```bash
git add skills/execute/SKILL.md
git commit -m "feat: add /adlc:execute lifecycle skill"
```

---

### Task 6: `/adlc:pr` skill (evolve `ship-pr`)

**Files:**
- Rename: `skills/ship-pr/SKILL.md` → `skills/pr/SKILL.md` (use `git mv`)
- Modify: the renamed `skills/pr/SKILL.md`

**Interfaces:**
- Consumes: `resolveCurrentTask`, `setStatus`, existing validate/push/PR logic. Precondition: on a task branch with completed work.
- Produces: an open PR auto-linked to the issue; status `In Review`.

- [ ] **Step 1: Rename the skill folder**

```bash
git mv skills/ship-pr skills/pr
```

- [ ] **Step 2: Update the skill**

Change frontmatter `name: ship-pr` → `name: pr`; keep the description trigger list and add "the ADLC `pr` lifecycle stage". Add to the workflow, after the PR is opened:
- Resolve the task via `resolveCurrentTask()` (parse the branch). If it resolves to a Linear issue, `setStatus(taskRef, "In Review")`; the branch name means Linear auto-links the PR.
- If no PM/Linear is configured or the branch doesn't map to an issue, skip the status step silently and behave as the generic PR opener (unchanged v0.1 behavior).

Keep all existing validation-first, no-secrets, branch-if-on-default rules.

- [ ] **Step 3: Validate**

Run: `test -f skills/pr/SKILL.md && ! test -d skills/ship-pr && grep -E '^name: pr$' skills/pr/SKILL.md && grep -c 'In Review\|resolveCurrentTask' skills/pr/SKILL.md`
Expected: folder renamed; name updated; Linear status step present.

- [ ] **Step 4: Commit**

```bash
git add -A skills/pr skills/ship-pr
git commit -m "feat: evolve ship-pr into the /adlc:pr lifecycle stage"
```

---

### Task 7: `/adlc:archive` skill + summary template

**Files:**
- Create: `skills/archive/SKILL.md`
- Create: `templates/archive-summary.md.template`

**Interfaces:**
- Consumes: `readTask`, `resolveCurrentTask`, `closeTask`. Precondition: the task's PR is merged.
- Produces: `docs/adlc/YYYY-MM-DD-<slug>.md` committed in the consumer repo; optional `## Outcome` on the issue; status `Done`.

- [ ] **Step 1: Write the template**

Create `templates/archive-summary.md.template` with sections: title, date, Linear issue link, PR link(s), **What shipped**, **Why**, **How** (approach in brief), **Notable deviations from the plan**. Curated summary — not a full spec/plan copy.

- [ ] **Step 2: Write the skill**

Frontmatter: `name: archive`; `description: Use when an ADLC task's PR has merged and the work is done — the final lifecycle stage. Writes a curated summary to the repo and closes the Linear issue.`

Body:
- **Purpose:** leave a durable, light record in the repo and close the task.
- **Workflow:**
  1. Resolve the task (`resolveCurrentTask`), `readTask`, and confirm the PR is merged; refuse otherwise.
  2. Render `${CLAUDE_PLUGIN_ROOT}/templates/archive-summary.md.template` (inline fallback) into `docs/adlc/YYYY-MM-DD-<slug>.md` — a curated summary (what/why/how + PR links + deviations), not a verbatim spec/plan.
  3. Commit the summary.
  4. Optionally `writeSection(taskRef, "Outcome", prLinks)`.
  5. Confirm with the user (gate), then `closeTask(taskRef)`.
- **Boundaries:** Ask First before closing the issue. Never copy the full spec/plan verbatim — keep it curated.
- **Done when:** the summary is committed and the issue is `Done`.

- [ ] **Step 3: Validate**

Run: `grep -E '^name: archive$' skills/archive/SKILL.md && grep -c 'docs/adlc/\|closeTask\|archive-summary' skills/archive/SKILL.md && test -f templates/archive-summary.md.template`
Expected: name matches; repo path + closeTask + template referenced; template exists.

- [ ] **Step 4: Commit**

```bash
git add skills/archive/SKILL.md templates/archive-summary.md.template
git commit -m "feat: add /adlc:archive lifecycle skill and summary template"
```

---

### Task 8: `/adlc:init` — PM config + Linear MCP detection

**Files:**
- Modify: `skills/init/SKILL.md`

**Interfaces:**
- Consumes: nothing new.
- Produces: `.adlc/config.json` (`{ "pm": { "provider": "linear", "team": "<key>" } }`) in the target repo; a `.adlc/`-ignored entry; a Linear-MCP presence check with setup guidance.

- [ ] **Step 1: Update the skill**

Add a "PM integration" step to the init workflow:
1. Ask which PM provider (default `linear`) and the Linear team key.
2. Write `.adlc/config.json` with `{ "pm": { "provider": "linear", "team": "<key>" } }`.
3. Ensure `.adlc/` is gitignored **except** `config.json` (config is shared; any task cache is not) — add `.adlc/*` + `!.adlc/config.json` to `.gitignore`.
4. Detect whether a Linear MCP server is available; if not, print exact setup instructions and mark the step incomplete rather than failing the whole init.
5. Mention the lifecycle skills (`/adlc:brainstorm … /adlc:archive`) in the closing summary.

- [ ] **Step 2: Validate**

Run: `grep -c '.adlc/config.json\|Linear MCP\|/adlc:brainstorm' skills/init/SKILL.md`
Expected: ≥3 (config path, MCP detection, lifecycle mention all present).

- [ ] **Step 3: Commit**

```bash
git add skills/init/SKILL.md
git commit -m "feat: add PM config and Linear MCP detection to /adlc:init"
```

---

### Task 9: README + reconcile v0.1 skills

**Files:**
- Modify: `README.md`
- Modify: `skills/write-spec/SKILL.md`

**Interfaces:**
- Consumes: all prior tasks (documents them).
- Produces: user-facing docs; a clear supersession note.

- [ ] **Step 1: Update README**

Add a "The ADLC lifecycle" section documenting `brainstorm → spec → plan → execute → pr → archive`, the Linear-as-live-workspace model, and the repo-summary archive. Update the skills table: add the six lifecycle skills; mark `write-spec` as the no-PM fallback; replace `ship-pr` with `pr`. Add a "Requires: Linear MCP" note.

- [ ] **Step 2: Add supersession note to write-spec**

At the top of `skills/write-spec/SKILL.md`, add a note: for repos using the Linear lifecycle, prefer `/adlc:spec` (writes to Linear); `/adlc:write-spec` remains for repos not using a PM system (writes a repo-local spec).

- [ ] **Step 3: Validate**

Run: `grep -c 'brainstorm\|/adlc:pr\|lifecycle' README.md && grep -c 'adlc:spec\|no-PM\|fallback' skills/write-spec/SKILL.md`
Expected: lifecycle documented in README; supersession note present.

- [ ] **Step 4: Commit**

```bash
git add README.md skills/write-spec/SKILL.md
git commit -m "docs: document the ADLC lifecycle and reconcile v0.1 skills"
```

---

### Task 10: End-to-end dogfood runbook + live validation

**Files:**
- Create: `docs/validation/2026-08-03-lifecycle-dogfood.md`

**Interfaces:**
- Consumes: the whole plugin.
- Produces: a repeatable manual validation runbook + a recorded first run.

- [ ] **Step 1: Write the runbook**

Create `docs/validation/2026-08-03-lifecycle-dogfood.md` describing a full cycle against a throwaway Linear issue: run each stage in order and assert, at each, the concrete observable: `## Idea` present after brainstorm; `## Specification` after spec; `## Plan` + `## Progress` and status `Todo` after plan; branch/worktree created, boxes ticking, status `In Progress` during execute; PR open, auto-linked, status `In Review` after pr; `docs/adlc/...md` committed and status `Done` after archive.

- [ ] **Step 2: Run the dogfood live**

Install the plugin locally (`/plugin marketplace add johngaynor/adlc` → `/plugin install adlc@adlc`), configure Linear MCP, and execute the runbook against a throwaway issue. Record pass/fail per assertion in the doc. Fix any skill that fails its assertion (amend the relevant task's file, re-commit) before proceeding.

Expected: every assertion passes, or failures are recorded with the follow-up fix.

- [ ] **Step 3: Commit**

```bash
git add docs/validation/2026-08-03-lifecycle-dogfood.md
git commit -m "test: add ADLC lifecycle dogfood runbook and first-run results"
```

---

## Self-Review

**1. Spec coverage** — every design section maps to a task:
- Lifecycle (6 stages) → Tasks 2–7. Data model + status + sections → Task 1 (contract) enforced by each skill. PM seam → Task 1. Task identity/parallelism → Task 1 (`resolveCurrentTask`) + Task 5 (branch/worktree at execute). Archive (manual, curated, `docs/adlc/`) → Task 7. Config + Linear MCP detection → Task 8. Error handling (preconditions, idempotent upsert, MCP-absent) → Task 1 preconditions table + per-skill precondition checks (Tasks 3–7) + Task 8 detection. Validation → Task 10. Self-contained / no-superpowers → Global Constraints + Task 2 (native brainstorm). Reconciliation of v0.1 skills → Tasks 6 (pr) + 9 (write-spec note). No gaps found.

**2. Placeholder scan** — no "TBD/TODO/implement later/handle edge cases". Each skill task specifies exact frontmatter, workflow steps, preconditions, ops used, boundaries, and done-condition. `<initials>/<slug>` etc. are format tokens, not placeholders.

**3. Type/name consistency** — operation names (`createTask`, `writeSection`, `readTask`, `tickPhase`, `setStatus`, `closeTask`, `findTasks`, `resolveCurrentTask`), section names (`## Idea/Specification/Plan/Progress/Outcome`), status values, and the branch format are defined once in Task 1 and used verbatim in Tasks 2–8. `ship-pr` → `pr` rename is consistent across Task 6 and Task 9.
