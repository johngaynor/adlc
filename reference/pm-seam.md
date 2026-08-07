# The PM Seam

Every ADLC lifecycle skill (`brainstorm`, `spec`, `plan`, `execute`, `pr`) and
planning-layer skill (`project`)
talks to the project-management system through this seam and nothing else — no skill
calls the PM ad hoc. The seam has exactly one implementation — **Linear**
(`"provider": "linear"`), connected per repo at `/adlc:init` and recorded as
`pm.provider` in `.adlc/config.json`. One Linear issue is the whole record for a
task, and its description is the canonical living document.

Skills speak in the operations and the canonical card layout below; the
concrete Linear MCP calls live in [Provider Mapping](#provider-mapping). Because
Linear is the seam's only backend, skills may also lean on Linear-native
constructs directly — `>>>` collapsibles, workflow states, milestones, branch
auto-linking — where a step needs them: the seam is an architectural discipline
(one PM touchpoint), not a lowest-common-denominator abstraction. If a second PM
ever becomes necessary, it would arrive as a new mapping behind these same
operations.

**No provider configured:** a stage whose preconditions require the PM (see
[Preconditions Contract](#preconditions-contract)) first verifies that
`.adlc/config.json` has a `pm` block with `"provider": "linear"` — any other
provider value is treated the same as no PM at all. If the check fails, the
stage refuses with: "No PM provider is configured for this repo — run
`/adlc:init` to connect one." Setup lives in exactly one place (`init`); no
other skill carries an inline setup flow.

## Hierarchy

Work lives at exactly one of four levels. These definitions are the seam's
shared vocabulary — planning-layer skills apply them as a sizing gate, and no
skill redefines them inline:

- **Initiative** — an outcome-phrased strategic goal served by several
  projects. Contains projects only, never issues. Its content is why + success
  criteria + health updates — no solution design. Creation is app-only
  (`/adlc:initiative` drafts the content); the seam can read initiatives and
  attach projects to them, but never create one.
- **Project** — one shippable chunk of work with a clear outcome and an end.
  Contains issues, documents, and milestones, and is designed to complete —
  progress graphs and predicted completion assume it. The **PRD** lives here:
  in the project description, with heavier supporting material as project
  documents. `/adlc:project` owns this level.
- **Milestone** — an ordered checkpoint inside one project, grouping a subset
  of its issues. Project-scoped — a milestone never spans projects.
- **Issue** — one task-sized change: the lifecycle card whose description is
  the canonical living document (see [Card Data Model](#card-data-model)).

The decision test, applied top-down:

- Needs **more than one shippable chunk** → initiative; exactly one → project.
- Has a **deliverable and an end** → project; a domain that never completes
  ("training", "billing") → a label, not a project.
- A **phase within one chunk** → milestone; something that spans chunks →
  initiative or label (milestones can't span projects).
- **One agent can spec → plan → PR it** → issue; anything that outgrows that →
  split it, or promote it a level.

## Operations

Fourteen operations make up the seam — nine at the issue tier, then five at the
planning tier (projects and initiatives). Each is documented below with its
signature and semantics; the concrete Linear MCP actions are tabulated under
[Provider Mapping](#provider-mapping).

### `createTask(title, brainstormMarkdown) → taskRef`

Create-or-retrofit. With no existing card: creates a new issue with the given
title, status `Backlog`, and description seeded with the brainstorm artifact — the
brief 1–2 sentence preamble plus its collapsed `>>> ### Notes` block (see
[Card Data Model](#card-data-model)). With an existing card (the caller passes
the card's `taskRef` in): replaces the description with that same layout, carrying
the old description's content into the Notes dropdown so nothing is lost. Returns
the `taskRef` (the issue identifier/URL) for the caller to hold in session context.

### `writeSection(taskRef, sectionName, markdown)`

Idempotent upsert of one artifact block in the issue's description. If the block
already exists, its content is replaced in place; if not, it is appended. Re-running
a stage never duplicates a block. Block boundaries per artifact (layout under
[Card Data Model](#card-data-model)):

- `Brainstorm` — from the start of the description to the first `---` divider (the
  untitled preamble plus its collapsed Notes block).
- `Specification` / `Technical plan` / `Progress` / `Outcome` — from the `---`
  divider immediately before the block's `# <name>` heading to the divider
  immediately before the next `# <name>` heading or, if none, the end of the
  description. A divider *inside* a block (like the one before the spec's
  Risks & unknowns dropdown) belongs to that block — boundaries are the dividers
  that directly precede an H1 heading.

A card still carrying legacy `## <name>` sections from before this layout is
migrated on touch: the first stage that would write one of these blocks rewrites
the legacy section into its H1-after-divider form.

### `readTask(taskRef) → { sections: {name→markdown}, checklist: [{text, done}], status }`

Reads the issue and parses its description into a map of artifact name → markdown
body using the block boundaries defined under `writeSection` — the untitled preamble
parses as `Brainstorm`, each `# <name>` block under its own name — plus the
`# Progress` checklist as a list of `{text, done}` entries, and the issue's current
status. This is how every stage
determines its precondition before acting.

### `tickPhase(taskRef, phaseText, done=true)`

Sets one checkbox in the `# Progress` block — the one whose text matches
`phaseText` — to checked (or unchecked, if `done=false`). Leaves every other line in
the section untouched.

### `applyLabel(taskRef, labelName)`

Idempotently ensures a label named `labelName` exists and is applied to the issue.
If the label does not exist in the backend, it is created first (default color).
Applying a label the issue already has is a no-op — a stage that re-runs never
duplicates a label or errors. Linear's label update replaces the issue's full
label set, so the mapping's append-before-write rule is mandatory (see
[Provider Mapping](#provider-mapping)).

### `setStatus(taskRef, status)`

Sets the issue's workflow state to one of the exact status values (see
[Status Mapping](#status-mapping)).

### `closeTask(taskRef)`

Sets the issue's status to `Done`. Equivalent to `setStatus(taskRef, "Done")`, named
separately because it is the lifecycle's terminal call — made by the post-merge
poller `pr` spawns when the PR merges, and by `/adlc:cleanup` as the safety net.

### `findTasks(filter) → taskRef[]`

Queries the PM for issues matching `filter` — for example, "has `# Specification`, no
`# Technical plan`" — so a stage can offer a resume candidate when the caller has no `taskRef`
in hand. "Brainstormed but not specced" means a non-empty description with no
`# Specification` heading; where the provider's search cannot express a filter
reliably, fall back to label presence (`spec`, `plan`) as the signal.

### `resolveCurrentTask() → taskRef`

Resolves the task for the current agent without a shared pointer. Pre-code stages
(`brainstorm`, `spec`, `plan`) use the `taskRef` already held in the agent's session
context. Code stages (`execute`, `pr`) parse the current git branch
(`<initials>/<issue-identifier>-<slug>`), extract the issue identifier, and look up the
issue. See [Task Identity Resolution](#task-identity-resolution).

### `saveProject(projectRef?, name, prdMarkdown, initiative?) → projectRef`

Create-or-retrofit, mirroring `createTask` one level up the
[Hierarchy](#hierarchy). With no `projectRef`: creates a project on the
configured team with the given name and the PRD as its description, attached to
`initiative` when one is given. With a `projectRef`: replaces that project's
description with the approved PRD and attaches the initiative if given. The PRD
always lives in the description — never only in a document. Returns the
`projectRef` (project URL or identifier) for the caller to hold in session
context; like `taskRef`, it is never persisted to a shared pointer file.

### `readProject(projectRef) → { name, description, initiative }`

Reads a project — its name, description (the PRD, or whatever freehand content
predates one), and initiative membership (empty if unattached). This is how
`/adlc:project` decides between retrofit input and a linkage prompt.

### `attachProjectDoc(projectRef, title, markdown)`

Creates a document on the project. Used only for supporting material too heavy
for the description — the PRD itself stays in the description per
`saveProject`.

### `listInitiatives() → initiativeRef[]`

Lists the workspace's initiatives, for a dialogue's "which initiative does this
serve?" question.

### `readInitiative(initiativeRef) → { name, description }`

Reads one initiative's content — its outcome statement and success criteria —
as context for downstream dialogue. There is deliberately no `createInitiative`:
initiative creation is app-only (see [Hierarchy](#hierarchy)).

## Card Data Model

One issue is the whole record for one task. Its description is a markdown
document that accumulates one artifact per lifecycle stage, every artifact in the
same glanceable layout — brief prose visible, detail tucked into collapsible
sections, blocks separated by `---` dividers, one H1 heading per block after
the untitled preamble. The layout is written in Linear-flavored markdown
(`>>>` collapsibles), which Linear renders natively:

```
Title:  <feature name>
Status: Backlog → Todo → In Progress → In Review → Done

Description:
  <brief 1–2 sentence task description>  ← brainstorm (untitled preamble)
  >>> ### Notes … >>>                    ← brainstorm (collapsed key findings)
  ---
  # Specification                        ← spec: summary paragraph ending in a verdict…
  >>> ### Approach … >>>                 ← …then its three fixed collapsibles
  >>> ### Blast radius … >>>
  >>> ### Test & coverage … >>>
  ---
  >>> ### Risks & unknowns … >>>         ← spec (blockers resolved before plan)
  ---
  # Technical plan                       ← written by plan
  ---
  # Progress                             ← written by plan (checklist), ticked by execute
  ---
  # Outcome                              ← written when a task closes without a code PR (optional)
```

The canonical artifacts, exact and case-sensitive where named, are:

- **`Brainstorm`** — the untitled preamble: 1–2 sentences that stand alone as a task
  description at a glance, plus a collapsed `>>> ### Notes` block whose bullets carry
  problem, motivation, and success criteria sharply enough that a fresh session could
  spec from the card alone.
- **`Specification`** — the `# Specification` H1 block: a small summary paragraph of
  what the spec discovered, the fixed collapsibles `### Approach`,
  `### Blast radius`, and `### Test & coverage`, then a `---` and a collapsed
  `>>> ### Risks & unknowns` block (see [Artifact skeletons](#artifact-skeletons)).
  Anything listed there as a blocker must be resolved before the technical plan.
- **`Technical plan`** — the `# Technical plan` H1 block: the phased plan written by
  `plan`.
- **`Progress`** — the `# Progress` H1 block: the phase checklist written by `plan`
  and ticked by `execute`.
- **`Outcome`** — the `# Outcome` H1 block, optional: written when a task closes
  without a code PR.

No skill invents an artifact outside this list.

`# Progress` holds a markdown checklist, one `- [ ] Phase <n>: <name>` line per
plan phase, its text matching the phase heading in `# Technical plan` exactly
(minus the `##`) — that text is `tickPhase`'s lookup key. `execute` flips boxes
to `- [x]` as phases complete via `tickPhase`. Phases live as a checklist inside
the issue rather than as sub-issues, to keep the whole task in one place.

`writeSection` upserts by the per-artifact block boundaries listed with the
operation above: it replaces its own artifact's block cleanly and leaves every other
block untouched. A stage that re-runs never duplicates its artifact or disturbs a
neighboring one.

Labels mirror artifact presence on the issue's card. `spec` and `plan` are the only
canonical labels — applied by their namesake stages via `applyLabel` the moment
their artifact lands (`# Specification` and `# Technical plan` + `# Progress`
respectively; the label keeps the name `plan` even though its section is titled
`# Technical plan`). They accumulate and are never removed: a rewritten spec keeps its `spec` label,
because the artifact still exists. As with section names, no skill invents a label
outside this list.

### Artifact skeletons

The `# Specification`, `# Technical plan`, and `# Progress` blocks each have
exactly one canonical skeleton, pinned here and nowhere else. A stage writing one
of these blocks reproduces its skeleton verbatim — same headings, same
collapsibles, same order — varying only content depth with the task. A section
with nothing to say states `None.`; it is never omitted, so any two cards are
structurally identical.

Two content rules, taken from the
[Linear Method](https://linear.app/method/write-issues-not-user-stories)'s
issue-writing lens, govern both artifacts:

- **Engineering-only.** A card is a plain-language engineering task with a
  clear, defined outcome. Product rationale and product decisions never appear
  in these blocks — they live upstream at the project/milestone. A product
  question surfaced mid-stage goes to `Risks & unknowns` and escalates; it is
  not settled on the card. The brainstorm preamble's 1–2 sentences remain the
  card's only "why".
- **Brevity.** Minimal necessary context; link out to deeper discussion
  instead of inlining it.

`# Specification` skeleton (written by `spec`):

```markdown
# Specification

<summary paragraph — reads standalone, absorbs the problem/goal, ends with an
explicit verdict: **Verdict: ready to plan.** or **Verdict: blocked on <X>.**>

>>> ### Approach

<decisions stated as decisions — "we will X"; real alternatives with honest
rejection reasons>

>>>

>>> ### Blast radius

<named files, modules, and contracts touched — plus what is deliberately
untouched>

>>>

>>> ### Test & coverage

<commitments execute can verify mechanically — named checks or commands>

>>>

---

>>> ### Risks & unknowns

<risks with mitigations; blockers flagged explicitly; `None.` if empty>

>>>
```

`# Technical plan` and `# Progress` skeletons (written by `plan`, ticked by
`execute`):

```markdown
# Technical plan

<one or two sentences on the phase ordering rationale>

## Phase 1: <name>

<plain-language prose/bullets of what changes>

**Verify:** <the mechanical check that proves this phase>

## Phase 2: <name>

…

---

# Progress

- [ ] Phase 1: <name>
- [ ] Phase 2: <name>
```

Every phase ends with its `**Verify:**` line, and every `# Progress` line
matches its `## Phase <n>: <name>` heading text exactly (minus the `##`) —
that exact match is `tickPhase`'s contract.

## Status Mapping

The exact status values, in lifecycle order:

```
Backlog → Todo → In Progress → In Review → Done
```

| Status | Set by |
|---|---|
| `Backlog` | `brainstorm` — set at issue creation (`createTask`) |
| `Todo` | `plan` — once `# Technical plan` and `# Progress` are written |
| `In Progress` | `execute` — once the task branch/worktree is created |
| `In Review` | `pr` — once the PR is open |
| `Done` | the post-merge poller `pr` spawns — once the PR merges (`closeTask`); `/adlc:cleanup` is the safety net |

`spec` does not change status; the issue stays `Backlog` while it gathers a
specification. Status is a stakeholder-visible progress signal as much as it is task
state — never skip a value or set one out of order. These five names are the seam's
canonical vocabulary; each is a real Linear workflow state (teams that renamed a
state map by position in the lifecycle order — see
[Provider Mapping](#provider-mapping)).

## Task Identity Resolution

Identity is per-agent and derived — there is no shared mutable "current task" pointer
anywhere in the repo, so parallel agents cannot collide.

- **Pre-code stages (`brainstorm`, `spec`, `plan`):** no branch exists yet, because no
  code has changed. The task's identity is the `taskRef` held in the invoking
  agent's own session context — returned by `createTask` in `brainstorm` and threaded
  forward. Cross-session resume (a different agent, or the same agent later) works by
  calling `findTasks` with a filter like "has `# Specification`, no `# Technical plan`", or by
  the user pasting the issue URL. There is no pointer file to read.
- **Code stages (`execute`, `pr`):** `execute` creates a git branch named
  `<initials>/<issue-identifier>-<slug>` in its own git worktree the moment code
  changes begin. The `<issue-identifier>` is the Linear issue identifier,
  lowercase (e.g. `jg/eng-142-linear-lifecycle`). From that
  point the branch is the per-agent task pointer: `resolveCurrentTask` parses the
  current branch name, extracts `<issue-identifier>`, and resolves it to the issue.
  This branch format is also what lets Linear auto-link the eventual PR back to
  the issue.
- **No shared pointer file, ever.** Any local cache a stage keeps for its own
  convenience (for example, a resolved `taskRef` written to disk to avoid re-parsing)
  is worktree-local and listed in that worktree's gitignore — it is never committed and
  never read by another agent.

## Preconditions Contract

Each stage reads the issue via `readTask` before acting and refuses to proceed if its
required section(s) are missing, so the lifecycle cannot be run out of order — with
one deliberate exception: `pr` is PM-optional. It must also work as a generic PR
opener with no PM configured at all, so it never hard-refuses on a missing
precondition. Instead, when a task *does* resolve, it treats `# Progress` as a
best-effort check: it warns and asks for confirmation if any box is unchecked,
rather than refusing outright — see `skills/pr/SKILL.md`.

Every stage below except `pr` also requires a configured provider — the `pm` block
check from the intro — and refuses with the standard message when it is absent.

| Stage | Requires (must already exist) |
|---|---|
| `brainstorm` | Nothing beyond a configured provider — this is the foundation stage; it creates the issue. |
| `spec` | The brainstorm artifact — a non-empty description with no `# Specification` heading yet. |
| `plan` | `# Specification`. |
| `execute` | `# Technical plan` and `# Progress`. |
| `pr` | PM-optional / best-effort: if a task resolves, warns (does not refuse) when any `# Progress` box is unchecked; if no task resolves, no precondition at all. |

## Provider Mapping

`/adlc:init` writes the connection details to `.adlc/config.json`. A skill reads
them once per run and follows the mapping below; nothing outside this section
may name a concrete PM API.

### Linear (`"provider": "linear"`)

Config shape (all values verified live at init):

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

- **Transport:** the Linear MCP server.
- **`taskRef`:** the Linear issue identifier (e.g. `ENG-142`) or issue URL.
- **`projectRef` / `initiativeRef`:** the Linear project / initiative ID, slug,
  or URL.
- **Card rendering:** the canonical layout verbatim — Linear renders `>>>`
  collapsibles natively.
- **Statuses:** real workflow states; `setStatus` updates the issue state to the
  canonical name. (Workspaces whose team renamed a state map by position in the
  lifecycle order, e.g. a team's "Ready for Dev" standing in for `Todo`.)
- **PR linking:** the task branch format is what lets Linear auto-link the PR to
  the issue; no PR-body reference is needed.

| Operation | Linear MCP action(s) |
|---|---|
| `createTask` | create issue, or read issue → update issue description |
| `writeSection` | update issue description |
| `readTask` | read issue |
| `tickPhase` | update description checkbox |
| `applyLabel` | list team labels → create team label (only if missing) → update issue labels to the **current set plus the new label**. The append-before-write is mandatory: Linear's label update replaces the issue's full label set, so writing only the new label would silently strip every label already on the issue. |
| `setStatus` | update issue state |
| `closeTask` | set state `Done` |
| `findTasks` | search issues |
| `resolveCurrentTask` | parse branch → look up issue by identifier |
| `saveProject` | create project (team from config), or update project description; initiative attach via the project's initiative fields |
| `readProject` | get project |
| `attachProjectDoc` | create project document |
| `listInitiatives` | list initiatives |
| `readInitiative` | get initiative |
