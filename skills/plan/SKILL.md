---
name: plan
description: Use when an ADLC task has an approved spec and needs its technical plan — after /adlc:spec. Writes a phased plan and a progress checklist into the Linear issue.
---

# Plan

Turn a task's `## Specification` into a phased technical plan and a live progress
checklist on the same Linear issue — the third stage of the ADLC lifecycle
(`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)).

## Arguments

- `taskRef` (optional) — the Linear issue URL or identifier to plan. If omitted:
  1. Use the `taskRef` already held in the session's context (set by a prior
     `/adlc:spec` in this same session).
  2. Otherwise call `findTasks(has "## Specification", no "## Plan")` per
     [`reference/pm-seam.md`](../../reference/pm-seam.md) and ask the user which
     candidate to plan — do not guess when more than one matches.

## Workflow

1. **Resolve the task.** Determine `taskRef` per the Arguments rule above.
2. **Read and gate on the precondition.** Call `readTask(taskRef)`. It must have
   `## Specification`. If it does not, refuse and tell the user to run `/adlc:spec`
   first — never draft a plan against an unspecified task.
3. **Break the work into phases.** Split `## Specification` into ordered,
   independently-committable phases. Each phase should be small enough to verify and
   commit on its own, and named clearly enough that its text can double as a
   checklist item later.
4. **Write the plan.** Call `writeSection(taskRef, "Plan", md)` per
   `reference/pm-seam.md` with the phase-by-phase plan — this upserts the
   `## Plan` block; re-running this stage overwrites it cleanly rather than
   duplicating it.
5. **Write the progress checklist.** Call `writeSection(taskRef, "Progress",
   checklist)` with one `- [ ]` line per phase, its text matching the phase name
   from `## Plan` exactly — `/adlc:execute` ticks these boxes by that same text.
6. **Advance the status.** Call `setStatus(taskRef, "Todo")` per the seam's Status
   Mapping — this is the exact point the lifecycle marks the task ready to be
   worked.
7. **Apply the `plan` label.** Call `applyLabel(taskRef, "plan")` per
   `reference/pm-seam.md` — this auto-creates the team label if missing and
   applies it idempotently, so the card signals the artifact at a glance.
8. **Present for approval.** Point the user at the Linear issue and get their
   explicit approval of the plan and phase breakdown. This gate happens **before**
   `/adlc:execute` runs — never let coding start on an unapproved plan.

## Boundaries

- **Never** start writing code, create a branch, or touch a worktree in this
  stage — that is `/adlc:execute`.
- **Ask First** before changing the agreed `## Specification` — if the plan
  surfaces a gap or a needed scope change, flag it to the user rather than quietly
  reshaping the spec.
- **Never** invent a Linear section name outside the five canonical ones in
  `reference/pm-seam.md`.

## Done when

The issue has `## Plan` and `## Progress` (one `- [ ]` per phase), carries the
`plan` label, its status is `Todo`, and the user has approved the plan.
