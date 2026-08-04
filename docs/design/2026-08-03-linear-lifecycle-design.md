# ADLC Task Lifecycle with Linear — Design

- **Date**: 2026-08-03
- **Status**: draft (approved for planning; refine later)
- **Scope**: The core ADLC process — how a unit of work moves from idea to shipped,
  with a product-management system (Linear first) as the live workspace and the repo
  as a light durable record.

## Goal

Give AI-assisted work a disciplined, resumable lifecycle where the **living
specification, technical plan, and progress sit on the team's PM system**
(stakeholder-visible, during the work) and a **curated summary lands in the repo
after completion** (durable engineering record). Linear is the first target; the
integration is built behind a thin seam so other PM systems can be added later.

ADLC is **self-contained** — it depends on no other Claude Code plugin. Every stage,
including the brainstorm dialogue, is a native ADLC skill.

## Relationship to prior art (open-mercato)

Open-mercato's spec system is entirely repo-native: specs are git files under
`.ai/specs/`, and their **lifecycle is directory location** (`.ai/specs/` →
`.ai/specs/implemented/` via `git mv`). The spec is the single source of truth the
whole way through.

ADLC keeps the same spec-first discipline but **splits the document's home by
phase**:

```
open-mercato:  repo(in-progress)      → repo(implemented/)
ADLC:          Linear (in-flight)     → repo (curated summary, after done)
```

The in-flight home moves to the PM tool (for stakeholder visibility and live
status), and the repo keeps a light, durable record instead of the full evolving
spec.

## The lifecycle

Six human-invoked stages. Each is a separate skill; the checkpoints between them are
the control surface. **All durable state lives in the Linear issue**, so any session
resumes by reading the issue.

```
brainstorm → spec → plan → execute → pr → archive
```

| Stage | Input | What it does | Linear effect | Human gate | Exit state |
|-------|-------|--------------|---------------|-----------|-----------|
| **brainstorm** | rough idea | Native collaborative-dialogue skill sharpens the idea, then **creates the issue** | New issue: title + `## Idea`; status `Backlog` | Approve the idea before the issue is created | Issue exists with idea |
| **spec** | issue (idea) | Researches and writes the specification | `## Specification` upserted; status stays `Backlog` | Review spec in Linear before planning | Issue has spec |
| **plan** | issue + spec | Produces the technical plan, broken into phases | `## Plan` + `## Progress` checklist upserted; status `Todo` | Approve the plan | Issue has plan + phases |
| **execute** | issue + plan | Creates the task branch/worktree, does the work phase-by-phase, ticks progress live | `## Progress` boxes ticked; status `In Progress` | Per-phase checkpoint | All phases done |
| **pr** | executed work | Runs validation, pushes, opens the PR (auto-linked to the issue via branch) | status `In Review` | Review / approve the PR | PR open & linked |
| **archive** | merged PR + issue | Writes the curated summary to the repo, commits it, closes the issue | status `Done` | Confirm before close | Repo summary committed; issue `Done` |

Status mapping doubles as the stakeholder-visible progress signal:
`Backlog → Todo → In Progress → In Review → Done`.

## Linear data model (issue-centric)

One Linear **issue is the whole record**. Its **description is the canonical living
document**, filled in progressively by the stages:

```
Title:  <feature name>
Status: Backlog → Todo → In Progress → In Review → Done

Description (markdown, section-based):
  ## Idea            ← brainstorm
  ## Specification   ← spec
  ## Plan            ← plan
  ## Progress        ← execute (markdown checklist, one box per phase, ticked live)
  ## Outcome         ← archive (PR link(s), shipped notes) — optional
```

- **Phases** are a markdown checklist in `## Progress` (`- [ ]` per phase). `execute`
  ticks boxes as it finishes. Chosen over sub-issues to stay lightweight and
  all-in-one-place, consistent with the issue-centric decision.
- Each stage reads the issue to determine what already exists (its precondition) and
  writes exactly one named `##` section (idempotent upsert).

## The PM seam (pluggability without over-building)

Skills never call Linear ad hoc. Every PM operation goes through one small, documented
set of operations. Linear implements them now; a future adapter (Notion, Jira)
implements the same list:

```
createTask(title, idea)          → taskRef
writeSection(taskRef, name, md)                    (idempotent upsert of a ## section)
readTask(taskRef)                → { sections, checklist, status }
tickPhase(taskRef, phase)
setStatus(taskRef, status)
closeTask(taskRef)
```

This is a thin seam, not a full abstraction framework — the full adapter abstraction
is deferred until a second PM system actually exists.

## Task identity and parallelism

Multiple agents work in parallel, each owning its own current task. Identity is
**per-agent and derived, never shared mutable state**. It splits by phase:

- **Pre-code stages (brainstorm / spec / plan):** no branch exists yet (no code
  changes). Identity is the **Linear issue ref held in the agent's session context**
  (returned when brainstorm created the issue). These stages only mutate their own
  distinct Linear issue and write nothing to a shared repo location, so running
  several in parallel cannot collide. Cross-session resume works by **querying Linear**
  (e.g. "issues with a spec but no plan") or pasting the issue URL — no local pointer
  file anywhere.
- **Code stages (execute / pr / archive):** `execute` creates the branch encoding the
  issue id (e.g. `jg/eng-142-<slug>`) **in its own git worktree** the moment real code
  changes begin. From here the **branch is the per-agent task pointer**, and
  worktree-per-task provides filesystem isolation for parallel execution. Linear
  auto-links the eventual PR to the issue via the branch name. `pr` and `archive`
  derive their issue from the branch.

Net model: **Linear issue = shared source of truth; git branch (per worktree) = each
agent's private handle to its own issue.** N agents → N worktrees → N branches → N
issues, fully independent.

## Archive

- **Trigger:** manual (`/adlc:archive`), run after the PR merges — matching the
  explicit-checkpoint style of every other stage. (Auto-triggering on merge via a
  webhook/hook is possible future work, out of scope for v1.)
- **Content:** a **curated summary** — what shipped, why, and how — plus links to the
  PR(s). Not a verbatim copy of the full spec/plan; the detailed record stays in
  Linear.
- **Location:** `docs/adlc/YYYY-MM-DD-<slug>.md` in the consumer repo — a dedicated
  archive directory, kept separate from any `.ai/specs/` tree.
- **Effect:** commit the summary, then close the Linear issue (`Done`).

## Error handling

- **Linear MCP absent or unauthenticated** → stages detect it up front and stop with
  setup instructions instead of half-writing state.
- **Issue not found / branch doesn't map to an open issue** → `execute` / `pr` /
  `archive` verify the issue ref resolves before acting, and refuse otherwise.
- **Partial writes** → `writeSection` is an idempotent upsert of a named section, so a
  re-run overwrites cleanly rather than duplicating.
- **Out-of-order stages** → each stage checks its precondition by reading the issue
  (e.g. `plan` refuses when there is no `## Specification`), so the pipeline cannot
  skip steps.

## Configuration

`/adlc:init` gains a PM section. It records `pm: { provider: linear, team: <key> }`
(in `.adlc/config.json` or a block in the scaffolded `CLAUDE.md`) and detects whether
the Linear MCP server is configured, printing setup instructions when it is not (the
Linear MCP server is not present by default).

## Validation

- **Dogfood against a real Linear team:** run the full `brainstorm → archive` cycle on
  a throwaway issue and confirm each section lands, progress ticks, the PR links, and
  the repo summary is written.
- **Per-skill pre/postcondition checks** exercise the `pm` seam end-to-end during the
  dogfood.
- No heavy unit harness for v1 — the seam is thin and the real risk is live
  integration behavior, which only a real run surfaces.

## Out of scope (future refinement)

- Full multi-provider PM adapter abstraction (only the thin seam now).
- Auto-triggering archive on PR merge (manual stage for v1).
- Enforcement hooks (e.g. block `execute` without an approved plan; SessionStart
  context injection).
- Sub-issue-based phase tracking.

## Dependency to configure

The Linear MCP server (or Linear API access) must be available to the agent; it is not
part of the default toolset. `/adlc:init` surfaces this.
