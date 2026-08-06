---
name: plan
description: Use when an ADLC task has an approved spec and needs its technical plan — after /adlc:spec. Writes a phased plan and a progress checklist into the task's card in the configured PM.
---

# Plan

Turn a task's `# Specification` into a phased technical plan and a live progress
checklist on the same issue — the third stage of the ADLC lifecycle
(`brainstorm → spec → plan → execute → pr`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)).

## Arguments

- `taskRef` (optional) — the issue URL or identifier to plan. If omitted:
  1. Use the `taskRef` already held in the session's context (set by a prior
     `/adlc:spec` in this same session).
  2. Otherwise call `findTasks(has "# Specification", no "# Technical plan")` per
     [`reference/pm-seam.md`](../../reference/pm-seam.md) and ask the user which
     candidate to plan — do not guess when more than one matches.

## Workflow

1. **Resolve the task.** Determine `taskRef` per the Arguments rule above.
2. **Read and gate on the precondition.** Call `readTask(taskRef)`. It must have
   `# Specification`. If it does not, refuse and tell the user to run `/adlc:spec`
   first — never draft a plan against an unspecified task. If the spec's
   `Risks & unknowns` block lists an unresolved blocker, surface it — blockers
   are resolved or explicitly accepted by the user before planning proceeds.
3. **Break the work into phases.** Split `# Specification` into ordered,
   independently-committable phases. Each phase should be small enough to verify and
   commit on its own, and named clearly enough that its text can double as a
   checklist item later. Phases are plain-language engineering tasks with a
   clear, defined outcome — the pinned skeleton's content rules
   (engineering-only, brevity) apply here as in the spec.
4. **Write the plan.** Call `writeSection(taskRef, "Technical plan", md)` per
   `reference/pm-seam.md`, following the pinned `# Technical plan` skeleton in
   its § Artifact skeletons exactly — a short ordering rationale, then one
   `## Phase <n>: <name>` heading per phase whose body says in plain language
   what changes and ends with a `**Verify:**` line naming that phase's
   mechanical check. Never invent or reshape the structure here. This upserts
   the `# Technical plan` block; re-running this stage overwrites it cleanly
   rather than duplicating it.
5. **Write the progress checklist.** Call `writeSection(taskRef, "Progress",
   checklist)` per the pinned `# Progress` skeleton — one
   `- [ ] Phase <n>: <name>` line per phase, its text matching the
   `## Phase <n>: <name>` heading exactly (minus the `##`) —
   `/adlc:execute` ticks these boxes by that same text.
6. **Advance the status.** Call `setStatus(taskRef, "Todo")` per the seam's Status
   Mapping — this is the exact point the lifecycle marks the task ready to be
   worked.
7. **Apply the `plan` label.** Call `applyLabel(taskRef, "plan")` per
   `reference/pm-seam.md` — this auto-creates the label if missing and
   applies it idempotently, so the card signals the artifact at a glance.
8. **Report and hand off.** Point the user at the issue and summarize the
   phase breakdown, then invoke the workflow bridge
   ([`/adlc:next`](../next/SKILL.md)) with `completedStage: plan` and the
   `taskRef`. The bridge's prompt *is* this stage's approval gate — and the
   last gate before the PR, since the execute→pr hop never prompts: choosing
   **Move to execute — continues through to PR** is the user's explicit
   approval of the plan, **Review first** shows the new `# Technical plan` and
   `# Progress` before deciding.

## Boundaries

- **Never** start writing code, create a branch, or touch a worktree in this
  stage — that is `/adlc:execute`.
- **Ask First** before changing the agreed `# Specification` — if the plan
  surfaces a gap or a needed scope change, flag it to the user rather than quietly
  reshaping the spec.
- **Never** invent an artifact outside the canonical layout in
  `reference/pm-seam.md`.
- **Never** let coding start on an unapproved plan — approval arrives through
  the bridge (**Move to execute**, or an explicit Run-to-PR horizon per the
  bridge's Boundaries), never by assumption.

## Done when

The issue has `# Technical plan` and `# Progress` matching the pinned skeletons
in `reference/pm-seam.md` § Artifact skeletons (one `- [ ]` per phase), carries the
`plan` label, its status is `Todo`, and the hand-off has passed through the
bridge — the user approved the plan (or their Run-to-PR horizon chained on), or
chose **Stop here** and the task rests cleanly at `Todo`.
