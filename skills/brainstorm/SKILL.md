---
name: brainstorm
description: Use when starting a new piece of work from a rough idea — the user wants to kick off an ADLC task. Runs a collaborative dialogue to sharpen the idea, then creates (or retrofits) the Linear card that will hold the whole task.
---

# Brainstorm

Sharpen a rough idea into a crisp problem statement, then create — or retrofit —
the task's Linear card — the first stage of the ADLC lifecycle (`brainstorm → spec
→ plan → execute → pr`, see [`METHODOLOGY.md`](../../METHODOLOGY.md)).
This is the only lifecycle stage that can run before a Linear card exists; every
later stage assumes one.

## Arguments

- `taskRef` (optional) — an existing Linear issue URL or identifier to brainstorm
  against. When given, this run is a **retrofit**: the card already exists (in any
  shape — an old-format `## Idea` card, a bare stub, someone's freehand notes) and
  this stage will replace its description with the brainstorm artifact, preserving
  the old content. When omitted, this run **creates** the card.

## Workflow

1. **Dialogue with the user.** Ask one question at a time to clarify purpose,
   constraints, and success criteria. Do not invent scope the user hasn't stated —
   if something is unclear, ask rather than assume. On a retrofit, read the
   existing card first (`readTask` per
   [`reference/pm-seam.md`](../../reference/pm-seam.md)) and treat its content as
   input to the dialogue.
2. **Draft the brainstorm artifact.** A concise title plus the artifact per the
   seam's Linear Data Model:
   - **Brief description** — 1–2 sentences, no heading, that stand alone as the
     task description at a glance.
   - **Collapsed Notes** — a `>>> ### Notes` block of key-finding bullets carrying
     the problem, the motivation/why-now, and success criteria sharply enough that
     a fresh session could spec from the card alone.

   Solution design and implementation detail do **not** belong here — that is
   spec/plan territory.
3. **Confirm with the user.** Present the title and artifact draft and get explicit
   approval. This gate happens **before** any card is created or overwritten —
   never touch the card on a draft the user hasn't seen.
4. **Create or retrofit the card.** Call `createTask(title, brainstormMarkdown)`
   per `reference/pm-seam.md`:
   - **No existing card:** creates the Linear issue with the approved title and
     artifact, status `Backlog` — no separate `setStatus` call is needed;
     `createTask` sets it per the seam's Status Mapping.
   - **Existing card (retrofit):** replaces the description with the approved
     artifact, carrying the old description's content into the Notes dropdown
     underneath the brief description so nothing is lost.
5. **Report and hand off.** Give the user the issue URL and hold its `taskRef` in
   session context — there is no shared pointer file (see `reference/pm-seam.md`
   § Task Identity Resolution). Then invoke the workflow bridge
   ([`/adlc:next`](../next/SKILL.md)) with `completedStage: brainstorm` and the
   `taskRef` — the bridge owns the lifecycle order and what comes next.

## Boundaries

- **Never** create or overwrite the Linear card before the user has approved the
  artifact draft.
- **Never** discard an existing card's content on a retrofit — it survives inside
  the Notes dropdown.
- **Never** write code, create a branch, or touch the task's worktree here — that
  starts at `/adlc:execute`.
- **Ask First** before broadening scope beyond what the user described.

## Done when

A `Backlog` Linear card exists (created, or retrofitted in place) whose description
is the brainstorm artifact — a brief 1–2 sentence description plus a collapsed
`### Notes` block — and the user has its URL.
