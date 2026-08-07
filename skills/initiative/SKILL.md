---
name: initiative
description: Use when the user wants to create or define an initiative — the strategic goal layer above projects ("create an initiative", "define an initiative", "new strategic goal"). Runs a sharpening dialogue and produces a paste-ready initiative Name + Description; creates nothing in Linear.
---

# Initiative

Sharpen an outcome-level goal into a paste-ready initiative definition — the
strategic layer above projects, where one goal is served by several shippable
chunks. This is a standalone utility skill, not a lifecycle stage: it runs the
defining dialogue and hands back the finished content, but creates nothing in
any PM — the Linear MCP server has no initiative operations, so the initiative
itself is created manually from the skill's output. Attaching contributing
projects and defining their PRDs happens downstream, not here.

## Arguments

- `goalSeed` (optional) — free text describing the rough goal, used to open the
  dialogue. When omitted, ask what goal the user has in mind.

## Workflow

1. **Open the dialogue.** Start from `goalSeed` (or ask for the goal) and
   proceed one question at a time, per the same dialogue discipline as
   `/adlc:brainstorm`: clarify, never invent scope the user hasn't stated.
2. **Apply the sizing gate.** Early on, ask the yes/no sizing question: does
   this goal plausibly need *more than one* shippable chunk of work to achieve?
   This is a sizing judgment only — do not enumerate the chunks. If the answer
   is single-chunk, recommend a plain project instead and offer to stop; the
   user may override and continue. The gate is soft — never refuse.
3. **Sharpen the three ingredients**, in whatever order the conversation
   naturally surfaces them, until each clears its bar:
   - **Outcome statement** — phrased as a state of the world, not an activity
     ("my coach can run my prep entirely through the app", not "improve coach
     features").
   - **Why now** — what hurts today, what this unlocks, why this beats waiting.
   - **Success criteria** — observable checks someone could verify true or
     false; not vibes.

   Stop there: no project breakdown, no solution design — those belong to the
   downstream stages.
4. **Draft against the template.** Render the ingredients into the format
   defined in [`template.md`](template.md) — Name plus templated Description —
   and present the draft for the user's approval. Iterate until approved.
5. **Deliver the approved draft.** Read the consuming repo's `.adlc/config.json`
   and branch on the same detection the lifecycle stages use:
   - **Linear configured** (`pm` block with `"provider": "linear"`): emit the
     draft as one copyable block, followed by the manual steps — enable
     Initiatives in workspace Settings → Initiatives if not already on, then
     create the initiative in the app and paste the content.
   - **No PM configured:** write the draft to
     `.ai/product/initiatives/<kebab-slug>.md` (slug derived from the Name),
     creating directories as needed, and report the path. This is the only
     file write this skill makes.
6. **Point downstream.** Close by noting the next step in the top-down flow:
   defining the contributing projects (the PRD stage), which is where chunks
   get enumerated and attached to the initiative.

## Boundaries

- **Never** write to Linear or invoke the PM seam — this skill's output is a
  draft, delivered as chat output or the no-PM fallback file, nothing else.
- **Never** enumerate the project breakdown or design solutions — the dialogue
  ends at the goal level.
- **Ask First** before widening scope beyond the goal the user described.

## Done when

The user has the approved draft in the templated format — as a paste-ready
block when Linear is configured, or written to
`.ai/product/initiatives/<kebab-slug>.md` when no PM is — and nothing has been
created or modified in any PM.
