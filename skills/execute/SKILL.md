---
name: execute
description: Use when an ADLC task is planned and it's time to write code — after /adlc:plan. Creates the task branch/worktree and works the plan phase-by-phase, ticking progress live in Linear.
---

# Execute

Turn a task's approved `## Plan` into committed code, one phase at a time, with
live progress ticked in the same Linear issue — the fourth stage of the ADLC
lifecycle (`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). This is the first stage where code
changes happen, and the first stage to have a git identity of its own: it is
where the task branch and its worktree are born.

## Arguments

- `taskRef` (optional) — the Linear issue URL or identifier to execute.

Resolve the task before anything else, in this order:

1. **Already inside a task worktree?** Check the current branch name against the
   branch format `<initials>/<issue-identifier>-<slug>` (see
   [`reference/pm-seam.md`](../../reference/pm-seam.md)). If it matches, this is a
   resumed run — call `resolveCurrentTask()` (parses the branch) to get `taskRef`
   and skip straight to [Workflow](#workflow) step 3; the branch/worktree already
   exists and must not be recreated.
2. **Not yet in a task worktree:** use the `taskRef` argument if given, otherwise
   the `taskRef` already held in the session's context (set by a prior
   `/adlc:plan` in this same session), otherwise call `findTasks(status "Todo",
   has "## Plan")` per `reference/pm-seam.md` and ask the user which candidate to
   execute — do not guess when more than one matches.

## Workflow

1. **Read and gate on the precondition.** Call `readTask(taskRef)`. It must have
   both `## Plan` and `## Progress`. If either is missing, refuse and tell the
   user to run `/adlc:plan` first — never start coding against an unplanned task.
2. **Create the branch and worktree.** This only happens once, on the first run
   for this task (see Arguments step 1 above):
   - Derive `<initials>` (the acting engineer's initials, lowercase — ask the
     user if they are not already known from git config), `<issue-identifier>`
     (the Linear issue's identifier, lowercase, e.g. `eng-142`), and `<slug>` (a
     short kebab-case slug from the issue title) to build the branch name
     `<initials>/<issue-identifier>-<slug>` exactly per
     [`reference/pm-seam.md`](../../reference/pm-seam.md).
   - Fetch the default branch and create a dedicated worktree off it, for
     example:
     ```bash
     git fetch origin
     git worktree add ../<repo-name>-<issue-identifier>-<slug> -b <initials>/<issue-identifier>-<slug> origin/main
     ```
     (substitute the repo's actual default branch if it isn't `main`). The
     worktree directory is a sibling of the current checkout, never inside it.
   - Move all further work in this run into that new worktree. This is what
     keeps parallel task execution isolated, and it is what lets `/adlc:pr` and
     `/adlc:archive` later resolve the same task by parsing the branch name.
3. **Advance the status.** Call `setStatus(taskRef, "In Progress")` per the
   seam's Status Mapping — do this once, right after the branch/worktree exists
   (or immediately, on a resumed run, if the issue isn't already `In Progress`).
4. **Work the plan phase by phase.** Re-read `## Progress` and take the phases in
   order. For each unchecked box:
   - **Implement** the phase exactly as `## Plan` describes it.
   - **Verify** it using the Validation Commands in the consuming repo's root
     `CLAUDE.md` (see [`METHODOLOGY.md`](../../METHODOLOGY.md) § 3 for what the
     label means) — the smallest relevant set for what the phase touched. A
     phase is not done until its verification passes; show the actual output,
     not a claim.
   - **Commit** the phase's work on the task branch with a message naming the
     phase, so the phase stays independently reviewable.
   - **Tick it.** Call `tickPhase(taskRef, phaseText)` per
     `reference/pm-seam.md`, using the exact phase text from `## Progress`.
   - **Checkpoint.** Report what was done and the verification evidence to the
     user before moving to the next phase, so they have a natural point to
     redirect, pause, or call out something the plan missed.
5. **Report and hand off when every box is checked.** Report the final state of
   `## Progress` and the branch it's committed on, then invoke the workflow
   bridge ([`/adlc:next`](../next/SKILL.md)) with `completedStage: execute` and
   the `taskRef`. Do **not** open a pull request from this stage — shipping is
   `/adlc:pr`, which the bridge offers as the next step.

## Boundaries

- **Never** run this skill's implementation work outside the task's own
  worktree once it exists — a resumed run always continues inside it, never in
  the original checkout or a new one.
- **Ask First** before deviating from the agreed `## Plan` — if a phase turns
  out to need a different approach than planned, flag it to the user before
  proceeding. If a deviation is approved, call it out plainly in that phase's
  commit message and in the checkpoint report so `/adlc:archive` can surface it
  later as a "notable deviation" — never let it pass silently.
- **Never** tick a phase (`tickPhase`) or move to the next one while its
  verification is failing.
- **Never** invent a Linear section name outside the five canonical ones in
  `reference/pm-seam.md`, and never open the PR from this stage.

## Done when

Every box in `## Progress` is checked, the issue's status is `In Progress`, and
all completed phases are committed on the task branch inside its own worktree.
