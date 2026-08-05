# The PM Seam

Every ADLC lifecycle skill (`brainstorm`, `spec`, `plan`, `execute`, `pr`)
talks to the project-management system through this seam and nothing else — no skill
calls the PM ad hoc. The seam has two implementations, chosen per repo at
`/adlc:init` and recorded as `pm.provider` in `.adlc/config.json`:

- **Linear** (`"provider": "linear"`) — one Linear issue is the whole record for a
  task.
- **GitHub Issues** (`"provider": "github"`) — one GitHub issue is the whole record
  for a task.

Either way, the issue's description/body is the canonical living document. Skills
speak only in the nine operations and the canonical card layout below; the concrete
API calls, status representation, and markdown rendering for each backend live in
[Provider Mappings](#provider-mappings), and a skill follows the mapping for the
configured provider. A future PM adapter (Notion, Jira, ...) would implement the
same nine operations as a third mapping; skills would not change. One caveat: the
canonical card layout is written in Linear-flavored markdown — `>>>` collapsible
sections — so every mapping must state how that layout renders in its backend, not
just how the operations map.

**No provider configured:** a stage whose preconditions require the PM (see
[Preconditions Contract](#preconditions-contract)) first verifies that
`.adlc/config.json` has a `pm` block. If it does not, the stage refuses with:
"No PM provider is configured for this repo — run `/adlc:init` to connect one."
Setup lives in exactly one place (`init`); no other skill carries an inline setup
flow.

## Operations

Nine operations make up the seam. Each is documented below with its signature and
semantics; the concrete backend actions per provider are tabulated under
[Provider Mappings](#provider-mappings).

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
duplicates a label or errors. The concrete create/apply semantics differ per
provider (see the mappings — Linear's label update replaces the full set, GitHub's
appends), and the mapping states how to apply safely.

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

## Card Data Model

One issue is the whole record for one task. Its description is a markdown
document that accumulates one artifact per lifecycle stage, every artifact in the
same glanceable layout — brief prose visible, detail tucked into collapsible
sections, blocks separated by `---` dividers, one H1 heading per block after
the untitled preamble. The layout is written here in Linear-flavored markdown
(`>>>` collapsibles); each provider mapping states how it renders in that backend:

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
  what the spec discovered, one `>>>` collapsible per topic, then a `---` and a
  collapsed `>>> ### Risks & unknowns` block. Anything listed there as a blocker must
  be resolved before the technical plan.
- **`Technical plan`** — the `# Technical plan` H1 block: the phased plan written by
  `plan`.
- **`Progress`** — the `# Progress` H1 block: the phase checklist written by `plan`
  and ticked by `execute`.
- **`Outcome`** — the `# Outcome` H1 block, optional: written when a task closes
  without a code PR.

No skill invents an artifact outside this list.

`# Progress` holds a markdown checklist, one `- [ ]` line per plan phase, with line
text matching the phase names from `# Technical plan`. `execute` flips boxes to `- [x]` as
phases complete via `tickPhase`. Phases live as a checklist inside the issue rather
than as sub-issues, to keep the whole task in one place.

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
canonical vocabulary; how each is represented in the backend (a Linear workflow
state, a GitHub `status:*` label or the issue's open/closed state) is defined per
provider under [Provider Mappings](#provider-mappings).

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
  changes begin. The `<issue-identifier>` is the provider's issue identity,
  lowercase — Linear's issue identifier (e.g. `jg/eng-142-linear-lifecycle`) or the
  bare GitHub issue number (e.g. `jg/142-linear-lifecycle`). From that
  point the branch is the per-agent task pointer: `resolveCurrentTask` parses the
  current branch name, extracts `<issue-identifier>`, and resolves it to the issue.
  How the eventual PR gets linked back to the issue is per provider: Linear
  auto-links from this branch format; GitHub links via `Closes #<number>` in the PR
  body (see the mappings).
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

## Provider Mappings

`/adlc:init` writes the provider choice to `.adlc/config.json`. A skill reads it
once per run and follows exactly one mapping below; nothing outside this section
may name a provider-specific API.

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

### GitHub Issues (`"provider": "github"`)

Config shape (owner/repo derived from the `origin` remote and verified live at
init):

```json
{
  "pm": {
    "provider": "github",
    "owner": "<owner>",
    "repo": "<repo>"
  }
}
```

- **Transport:** the GitHub MCP server when configured; otherwise the `gh` CLI
  (`gh issue view/edit/list`, `gh api`) — the two are interchangeable here, and
  `pr`/`cleanup` already shell out to `gh`.
- **`taskRef`:** the issue number (`#142`) or issue URL, always within the
  configured `owner`/`repo`.
- **Card rendering:** GitHub markdown has no `>>>` collapsibles; every
  `>>> ### <title> … >>>` block renders as
  `<details><summary><b><title></b></summary>` … `</details>`, with a blank line
  after `<summary>` and before `</details>` so the markdown inside still renders.
  Everything else — `---` dividers, one H1 per block, artifact names, boundary
  rules — is identical, and `writeSection`/`readTask` use the same
  divider-before-H1 boundaries on the issue body.
- **Statuses:** GitHub issues are natively only open/closed. `Backlog`, `Todo`,
  `In Progress`, and `In Review` are mutually-exclusive repo labels
  `status:backlog`, `status:todo`, `status:in-progress`, `status:in-review` —
  `setStatus` removes the current `status:*` label and adds the new one, creating
  it on first use. `Done` is the issue's closed state (close as completed), not a
  label; `closeTask` also removes any lingering `status:*` label.
- **PR linking:** GitHub does not auto-link from branch names; `pr` puts
  `Closes #<number>` in the PR body. Merging then closes the issue natively, so
  the post-merge poller's `closeTask` verifies (and finishes the label cleanup)
  rather than being the sole closer.

| Operation | GitHub action(s) (MCP or `gh`) |
|---|---|
| `createTask` | create issue with `status:backlog` label, or read issue → update issue body |
| `writeSection` | read issue body → update issue body |
| `readTask` | read issue (body, labels, open/closed state) |
| `tickPhase` | read issue body → update the one checkbox → update issue body |
| `applyLabel` | add label to issue (GitHub's add-label call appends and auto-creates the repo label — no read-modify-write needed) |
| `setStatus` | remove old `status:*` label → add new `status:*` label |
| `closeTask` | close issue (reason: completed) → remove `status:*` label |
| `findTasks` | search issues (`repo:<owner>/<repo> is:issue` + `in:body` text); fall back to label presence (`spec`, `plan`, `status:*`) where body search is too fuzzy |
| `resolveCurrentTask` | parse the leading `<number>` after `<initials>/` → look up issue `#<number>` |
