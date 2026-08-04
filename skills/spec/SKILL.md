---
name: spec
description: Use when an ADLC task has an idea and needs its specification written — after /adlc:brainstorm. Researches and writes the spec into the task's Linear issue.
---

# Spec

Turn a task's `## Idea` into a reviewed specification on the same Linear issue — the
second stage of the ADLC lifecycle (`brainstorm → spec → plan → execute → pr →
archive`, see [`METHODOLOGY.md`](../../METHODOLOGY.md)).

## Arguments

- `taskRef` (optional) — the Linear issue URL or identifier to spec. If omitted:
  1. Use the `taskRef` already held in the session's context (set by a prior
     `/adlc:brainstorm` in this same session).
  2. Otherwise call `findTasks(has "## Idea", no "## Specification")` per
     [`reference/pm-seam.md`](../../reference/pm-seam.md) and ask the user which
     candidate to spec — do not guess when more than one matches.

## Workflow

1. **Resolve the task.** Determine `taskRef` per the Arguments rule above.
2. **Read and gate on the precondition.** Call `readTask(taskRef)`. It must have
   `## Idea`. If it does not, refuse and tell the user to run `/adlc:brainstorm`
   first — never draft a specification from scratch without an approved idea.
3. **Research.** Use the consuming repo's own Task Router (see
   [`METHODOLOGY.md`](../../METHODOLOGY.md) § 2) to find the guides, modules, and
   existing specs relevant to `## Idea`, and read them before drafting. Do not
   invent architecture the repo's own conventions already answer.
4. **Draft the specification.** Cover, at minimum:
   - **Problem / goal** — restated from `## Idea`, sharpened by research.
   - **Approach** — the proposed design, the alternatives considered and why they
     were rejected, and the blast radius (files, modules, and contracts touched).
   - **Test / coverage plan** — what the work will need to prove it's correct.
5. **Write it back.** Call `writeSection(taskRef, "Specification", md)` per
   `reference/pm-seam.md` — this upserts the `## Specification` block; re-running
   this stage overwrites it cleanly rather than duplicating it.
6. **Apply the `spec` label.** Call `applyLabel(taskRef, "spec")` per
   `reference/pm-seam.md` — this auto-creates the team label if missing and
   applies it idempotently, so the card signals the artifact at a glance.
7. **Report and hand off.** Point the user at the Linear issue and summarize the
   key design decisions, then invoke the workflow bridge
   ([`/adlc:next`](../next/SKILL.md)) with `completedStage: spec` and the
   `taskRef`. The bridge's prompt *is* this stage's review gate: choosing
   **Move to plan** is the user's explicit approval of the specification,
   **Review first** shows the new `## Specification` before deciding.

## Boundaries

- **Ask First** before expanding scope beyond what `## Idea` describes — new
  requirements go back through `/adlc:brainstorm`, not silently into the spec.
- **Never** start the plan or write code in this stage — that is `/adlc:plan` and
  `/adlc:execute`.
- **Never** invent a Linear section name outside the five canonical ones in
  `reference/pm-seam.md`.
- **Never** let planning start on an unreviewed spec — approval arrives through
  the bridge (**Move to plan**, or an explicit auto-mode opt-in per the bridge's
  Boundaries), never by assumption.

## Done when

The issue has a `## Specification` section, carries the `spec` label, and the
hand-off has passed through the bridge — the user approved it (or auto mode
chained on), or chose **Stop here** and the task rests cleanly at `Backlog`.
