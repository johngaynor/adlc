---
name: add-milestone
description: Use when the user wants to add a milestone to a Linear project — "add a milestone", "create a checkpoint", "add a beta milestone to X". Creates one ordered checkpoint (name, optional target date, phase-level description) after a confirm gate.
---

# Add a milestone

Create one **milestone** — an ordered checkpoint inside one project, per the
hierarchy defined in [`reference/pm-seam.md`](../../reference/pm-seam.md)
§ Hierarchy — with a name, an optional target date, and a phase-level
description; nothing more. This is a planning-layer utility skill: the
primitive the breakdown stage will invoke once per phase, and usable standalone
on any project at any time. It talks to the PM through the seam's
planning-tier operations only, and creates no issues — filing issues into
milestones is the lifecycle's and the breakdown stage's job.

## Arguments

- `$ARGUMENTS` (optional, freeform) — any of the milestone's fields the user
  already knows: the target **project** (URL, ID, or name), the milestone
  **name**, a **target date**, and a **description**. Everything is optional at
  invocation; the workflow prompts only for what's missing.

## Workflow

1. **Resolve the project.** An explicit project in `$ARGUMENTS` wins.
   Otherwise default to `.adlc/config.json`'s `pm.project` (when `pm.provider`
   is `"linear"` — the same detection the lifecycle stages use) and confirm
   that default with the user as part of step 4's draft. With no explicit
   project and no configured provider, refuse with the seam's standard
   message: "No PM provider is configured for this repo — run `/adlc:init` to
   connect one."
2. **Pre-flight the checkpoint order.** Call `listMilestones(projectRef)` per
   `reference/pm-seam.md` and show the project's existing milestones in order,
   so the user sees where the new checkpoint lands (always at the end — the
   seam exposes no reordering). If a milestone with the same name already
   exists, ask whether they meant that one — `createMilestone` is create-only,
   and a same-name hit is a question, never a silent update.
3. **Gather the missing fields, holding the content bar.** Prompt only for
   what `$ARGUMENTS` didn't provide:
   - **Name** (required) — 1–4 words, outcome-flavored: a state the project
     reaches ("Beta", "Coach can view"), not a task.
   - **Description** (encouraged) — 1–3 sentences of exit criteria: what
     "done" means for this phase. Push anything longer down to issues —
     detail lives at the issue level per § Hierarchy, and spec-level content
     is declined, not stored here.
   - **Target date** (optional) — a single ISO date; skip freely, it can be
     set in Linear later.
4. **Confirm the draft.** Present the project, name, target date, description,
   and where the checkpoint lands in the existing order, and get explicit
   approval. Never touch the PM before this approval.
5. **Create it.** Call `createMilestone(projectRef, name, targetDate?,
   description?)` per `reference/pm-seam.md`.
6. **Report.** Give the user the milestone's URL, constructed per the seam's
   Provider Mapping (`<project-url>/milestone/<milestoneRef>`), and note that
   it sits at 0% until issues are filed into it.

## Boundaries

- **Never** create the milestone before the user approves the draft.
- **Never** update, rename, or reorder an existing milestone — this skill is
  create-only; a same-name match becomes a question (step 2), not an edit.
- **Never** accept spec-level detail into the description — the milestone
  carries a phase-level description only; detail belongs on its issues.
- **Never** create issues or assign issues to the milestone here — that is the
  lifecycle's and the breakdown stage's job.
- This is not a lifecycle stage: no task card, no status changes, no
  `/adlc:next` hand-off. One milestone per run — deriving a project's full
  phase skeleton is the breakdown stage; repeat invocations are fine.

## Done when

The approved milestone exists in the target project with its name, optional
target date, and phase-level description, and the user has its URL. Nothing
else changed — no issues touched, no existing milestones altered.
