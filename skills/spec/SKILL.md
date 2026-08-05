---
name: spec
description: Use when an ADLC task has been brainstormed and needs its specification written — after /adlc:brainstorm. Researches and writes the spec into the task's Linear issue.
---

# Spec

Turn a task's brainstorm artifact into a reviewed specification on the same Linear
card — the second stage of the ADLC lifecycle (`brainstorm → spec → plan → execute
→ pr → archive`, see [`METHODOLOGY.md`](../../METHODOLOGY.md)).

## Arguments

- `taskRef` (optional) — the Linear issue URL or identifier to spec. If omitted:
  1. Use the `taskRef` already held in the session's context (set by a prior
     `/adlc:brainstorm` in this same session).
  2. Otherwise call `findTasks(brainstormed but not specced)` per
     [`reference/pm-seam.md`](../../reference/pm-seam.md) — a non-empty
     description with no `# Specification` heading (fall back to label absence
     where search can't express that) — and ask the user which candidate to
     spec; do not guess when more than one matches.

## Workflow

1. **Resolve the task.** Determine `taskRef` per the Arguments rule above.
2. **Read and gate on the precondition.** Call `readTask(taskRef)`. It must have
   the brainstorm artifact — a non-empty description (the brief summary, normally
   with its collapsed Notes). If the description is empty, refuse and tell the
   user to run `/adlc:brainstorm` first — never draft a specification from
   scratch without an approved brainstorm. A card that already carries a
   `# Specification` block is a re-run: the block will be replaced cleanly.
3. **Research.** Use the consuming repo's own Task Router (see
   [`METHODOLOGY.md`](../../METHODOLOGY.md) § 2) to find the guides, modules, and
   existing specs relevant to the brainstorm artifact, and read them before
   drafting. Do not invent architecture the repo's own conventions already
   answer.
4. **Draft the specification** per the seam's Linear Data Model layout:
   - **Summary paragraph** — a small paragraph, directly under the
     `# Specification` heading, of what the specification discovered.
   - **Topic collapsibles** — one `>>>` block per topic. Cover at minimum the
     approach (proposed design, alternatives considered and why rejected) and
     the blast radius (files, modules, contracts touched), plus a test/coverage
     plan for what the work must prove.
   - **Risks & unknowns** — after a `---` divider, a collapsed
     `>>> ### Risks & unknowns` block. Anything that is a blocker must be
     surfaced here and addressed before the task moves to `/adlc:plan`.
5. **Write it back.** Call `writeSection(taskRef, "Specification", md)` per
   `reference/pm-seam.md` — this upserts the `# Specification` block by the
   seam's boundary rules; re-running this stage overwrites it cleanly rather
   than duplicating it, and never disturbs the brainstorm preamble above it.
6. **Apply the `spec` label.** Call `applyLabel(taskRef, "spec")` per
   `reference/pm-seam.md` — this auto-creates the team label if missing and
   applies it idempotently, so the card signals the artifact at a glance.
7. **Report and hand off.** Point the user at the Linear issue and summarize the
   key design decisions and any open risks, then invoke the workflow bridge
   ([`/adlc:next`](../next/SKILL.md)) with `completedStage: spec` and the
   `taskRef`. The bridge's prompt *is* this stage's review gate: choosing
   **Move to plan** is the user's explicit approval of the specification,
   **Review first** shows the new `# Specification` before deciding.

## Boundaries

- **Ask First** before expanding scope beyond what the brainstorm artifact
  describes — new requirements go back through `/adlc:brainstorm`, not silently
  into the spec.
- **Never** start the plan or write code in this stage — that is `/adlc:plan` and
  `/adlc:execute`.
- **Never** let a known blocker ride silently into planning — blockers live in
  `Risks & unknowns` and are resolved (or explicitly accepted by the user)
  before `/adlc:plan`.
- **Never** invent an artifact outside the canonical layout in
  `reference/pm-seam.md`.
- **Never** let planning start on an unreviewed spec — approval arrives through
  the bridge (**Move to plan**, or an explicit auto-mode opt-in per the bridge's
  Boundaries), never by assumption.

## Done when

The card has a `# Specification` block (summary paragraph, topic collapsibles,
and a Risks & unknowns dropdown), carries the `spec` label, and the hand-off has
passed through the bridge — the user approved it (or auto mode chained on), or
chose **Stop here** and the task rests cleanly at `Backlog`.
