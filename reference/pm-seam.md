# The PM Seam

Every ADLC lifecycle skill (`brainstorm`, `spec`, `plan`, `execute`, `pr`)
talks to the project-management system through this seam and nothing else — no skill
calls Linear ad hoc. Linear is the only implementation for v1: one Linear issue is the
whole record for a task, and its description is the canonical living document. A future
PM adapter (Notion, Jira, ...) would implement the same nine operations against a
different backend; skills would not change. One caveat: the card layout itself (see
[Linear Data Model](#linear-data-model)) uses Linear-flavored markdown — `>>>`
collapsible sections — so an adapter must translate the layout, not just the
operations.

## Operations

Nine operations make up the seam. Each is documented below with its signature, what
it does, and the concrete Linear MCP action it maps to.

### `createTask(title, brainstormMarkdown) → taskRef`

Create-or-retrofit. With no existing card: creates a new Linear issue with the given
title, status `Backlog`, and description seeded with the brainstorm artifact — the
brief 1–2 sentence preamble plus its collapsed `>>> ### Notes` block (see
[Linear Data Model](#linear-data-model)). With an existing card (the caller passes
the card's `taskRef` in): replaces the description with that same layout, carrying
the old description's content into the Notes dropdown so nothing is lost. Returns
the `taskRef` (the issue identifier/URL) for the caller to hold in session context.

- **Linear MCP actions:** create issue, or read issue → update issue description.

### `writeSection(taskRef, sectionName, markdown)`

Idempotent upsert of one artifact block in the issue's description. If the block
already exists, its content is replaced in place; if not, it is appended. Re-running
a stage never duplicates a block. Block boundaries per artifact (layout under
[Linear Data Model](#linear-data-model)):

- `Brainstorm` — from the start of the description to the first `---` divider (the
  untitled preamble plus its collapsed Notes block).
- `Specification` — from the `---` divider immediately before the `# Specification`
  heading to the next `## <name>` heading (`## Plan`, `## Progress`, `## Outcome`)
  or, if none exists yet, the end of the description.
- `Plan` / `Progress` / `Outcome` — legacy `##` sections: from the `## <name>`
  heading line to the next `##` heading or end of description.

- **Linear MCP action:** update issue description.

### `readTask(taskRef) → { sections: {name→markdown}, checklist: [{text, done}], status }`

Reads the issue and parses its description into a map of artifact name → markdown
body using the block boundaries defined under `writeSection` — the untitled preamble
parses as `Brainstorm`, the `# Specification` block as `Specification`, the legacy
`##` sections under their own names — plus the `## Progress` checklist as a list of
`{text, done}` entries, and the issue's current status. This is how every stage
determines its precondition before acting.

- **Linear MCP action:** read issue.

### `tickPhase(taskRef, phaseText, done=true)`

Sets one checkbox in the `## Progress` section — the one whose text matches
`phaseText` — to checked (or unchecked, if `done=false`). Leaves every other line in
the section untouched.

- **Linear MCP action:** update description checkbox.

### `applyLabel(taskRef, labelName)`

Idempotently ensures a team label named `labelName` exists and is applied to the
issue. First look the label up by name on the issue's team; if it does not exist,
create it team-level (Linear's default color). Then update the issue's labels to
its **current set plus the new label**. The append-before-write is mandatory: the
Linear MCP's label update replaces the issue's full label set, so writing only the
new label would silently strip every label already on the issue. Applying a label
the issue already has is a no-op — a stage that re-runs never duplicates a label
or errors.

- **Linear MCP actions:** list labels → create label (only if missing) → update
  issue labels.

### `setStatus(taskRef, status)`

Sets the issue's workflow state to one of the exact status values (see
[Status Mapping](#status-mapping)).

- **Linear MCP action:** update issue state.

### `closeTask(taskRef)`

Sets the issue's status to `Done`. Equivalent to `setStatus(taskRef, "Done")`, named
separately because it is the lifecycle's terminal call — made by the post-merge
poller `pr` spawns when the PR merges, and by `/adlc:cleanup` as the safety net.

- **Linear MCP action:** set state `Done`.

### `findTasks(filter) → taskRef[]`

Queries Linear for issues matching `filter` — for example, "has `# Specification`, no
`## Plan`" — so a stage can offer a resume candidate when the caller has no `taskRef`
in hand. "Brainstormed but not specced" means a non-empty description with no
`# Specification` heading; where Linear's search cannot express a filter reliably,
fall back to label presence (`spec`, `plan`) as the signal.

- **Linear MCP action:** search issues.

### `resolveCurrentTask() → taskRef`

Resolves the task for the current agent without a shared pointer. Pre-code stages
(`brainstorm`, `spec`, `plan`) use the `taskRef` already held in the agent's session
context. Code stages (`execute`, `pr`) parse the current git branch
(`<initials>/<issue-identifier>-<slug>`), extract the issue identifier, and look up the
issue. See [Task Identity Resolution](#task-identity-resolution).

- **Linear MCP action:** parse branch → issue.

## Linear Data Model

One Linear issue is the whole record for one task. Its description is a markdown
document that accumulates one artifact per lifecycle stage. The two early artifacts
use a glanceable layout — brief prose visible, detail tucked into Linear `>>>`
collapsibles, stages separated by `---` dividers — while the later artifacts keep
plain `##` sections:

```
Title:  <feature name>
Status: Backlog → Todo → In Progress → In Review → Done

Description:
  <brief 1–2 sentence task description>  ← brainstorm (untitled preamble)
  >>> ### Notes … >>>                    ← brainstorm (collapsed key findings)
  ---
  # Specification                        ← spec: small summary paragraph…
  >>> ### <topic> … >>> (one per topic)  ← …with detail per collapsible
  ---
  >>> ### Risks & unknowns … >>>         ← spec (blockers resolved before plan)
  ---
  ## Plan                                ← written by plan
  ## Progress                            ← written by plan (checklist), ticked by execute
  ## Outcome                             ← written when a task closes without a code PR (optional)
```

The canonical artifacts, exact and case-sensitive where named, are:

- **`Brainstorm`** — the untitled preamble: 1–2 sentences that stand alone as a task
  description at a glance, plus a collapsed `>>> ### Notes` block whose bullets carry
  problem, motivation, and success criteria sharply enough that a fresh session could
  spec from the card alone.
- **`Specification`** — the `# Specification` H1 block: a small summary paragraph of
  what the spec discovered, one `>>>` collapsible per topic, then a `---` and a
  collapsed `>>> ### Risks & unknowns` block. Anything listed there as a blocker must
  be resolved before the technical plan.
- **`## Plan`, `## Progress`, `## Outcome`** — legacy `##` sections, unchanged.

No skill invents an artifact outside this list.

`## Progress` holds a markdown checklist, one `- [ ]` line per plan phase, with line
text matching the phase names from `## Plan`. `execute` flips boxes to `- [x]` as
phases complete via `tickPhase`. Phases live as a checklist inside the issue rather
than as Linear sub-issues, to keep the whole task in one place.

`writeSection` upserts by the per-artifact block boundaries listed with the
operation above: it replaces its own artifact's block cleanly and leaves every other
block untouched. A stage that re-runs never duplicates its artifact or disturbs a
neighboring one.

Labels mirror artifact presence on the issue's card. `spec` and `plan` are the only
canonical labels — applied by their namesake stages via `applyLabel` the moment
their artifact lands (`# Specification` and `## Plan` + `## Progress` respectively).
They accumulate and are never removed: a rewritten spec keeps its `spec` label,
because the artifact still exists. As with section names, no skill invents a label
outside this list.

## Status Mapping

The exact status values, in lifecycle order:

```
Backlog → Todo → In Progress → In Review → Done
```

| Status | Set by |
|---|---|
| `Backlog` | `brainstorm` — set at issue creation (`createTask`) |
| `Todo` | `plan` — once `## Plan` and `## Progress` are written |
| `In Progress` | `execute` — once the task branch/worktree is created |
| `In Review` | `pr` — once the PR is open |
| `Done` | the post-merge poller `pr` spawns — once the PR merges (`closeTask`); `/adlc:cleanup` is the safety net |

`spec` does not change status; the issue stays `Backlog` while it gathers a
specification. Status is a stakeholder-visible progress signal as much as it is task
state — never skip a value or set one out of order.

## Task Identity Resolution

Identity is per-agent and derived — there is no shared mutable "current task" pointer
anywhere in the repo, so parallel agents cannot collide.

- **Pre-code stages (`brainstorm`, `spec`, `plan`):** no branch exists yet, because no
  code has changed. The task's identity is the Linear `taskRef` held in the invoking
  agent's own session context — returned by `createTask` in `brainstorm` and threaded
  forward. Cross-session resume (a different agent, or the same agent later) works by
  calling `findTasks` with a filter like "has `# Specification`, no `## Plan`", or by
  the user pasting the issue URL. There is no pointer file to read.
- **Code stages (`execute`, `pr`):** `execute` creates a git branch named
  `<initials>/<issue-identifier>-<slug>` (for example `jg/eng-142-linear-lifecycle`) in
  its own git worktree the moment code changes begin. From that point the branch is the
  per-agent task pointer: `resolveCurrentTask` parses the current branch name, extracts
  `<issue-identifier>`, and resolves it to the issue. The branch format is also what
  lets Linear auto-link the eventual PR to the issue.
- **No shared pointer file, ever.** Any local cache a stage keeps for its own
  convenience (for example, a resolved `taskRef` written to disk to avoid re-parsing)
  is worktree-local and listed in that worktree's gitignore — it is never committed and
  never read by another agent.

## Preconditions Contract

Each stage reads the issue via `readTask` before acting and refuses to proceed if its
required section(s) are missing, so the lifecycle cannot be run out of order — with
one deliberate exception: `pr` is PM-optional. It must also work as a generic PR
opener with no Linear/PM configured at all, so it never hard-refuses on a missing
precondition. Instead, when a task *does* resolve, it treats `## Progress` as a
best-effort check: it warns and asks for confirmation if any box is unchecked,
rather than refusing outright — see `skills/pr/SKILL.md`.

| Stage | Requires (must already exist) |
|---|---|
| `brainstorm` | Nothing — this is the foundation stage; it creates the issue. |
| `spec` | The brainstorm artifact — a non-empty description with no `# Specification` heading yet. |
| `plan` | `# Specification`. |
| `execute` | `## Plan` and `## Progress`. |
| `pr` | PM-optional / best-effort: if a task resolves, warns (does not refuse) when any `## Progress` box is unchecked; if no task resolves, no precondition at all. |
