---
name: next
description: Use when an ADLC lifecycle stage has just finished and the task should hand off to whatever stage comes next, or to resume an in-flight task at its next step — presents Move to next / Run to PR / Review first / Stop here; a chosen Run-to-PR horizon chains the task's remaining stages automatically, and the execute→pr hop never prompts.
---

# Next (the workflow bridge)

Hand a task off from the lifecycle stage that just finished to the one that comes
next — the connective tissue of the ADLC lifecycle (`brainstorm → spec → plan →
execute → pr`, see [`METHODOLOGY.md`](../../METHODOLOGY.md)). Every
stage with a successor invokes this skill as its final step, and it can also be
invoked standalone to resume a task at whatever comes next. The bridge is the
single owner of the lifecycle order: no stage names its own successor. It only
reads the task (via `readTask` / `findTasks` per
[`reference/pm-seam.md`](../../reference/pm-seam.md)) — it never writes to it.

## Arguments

Both optional. An invoking stage passes both; a standalone invocation may pass
neither.

- `taskRef` — the Linear issue URL or identifier. If omitted: use the `taskRef`
  already held in the session's context; otherwise call `resolveCurrentTask()`
  (parses a task branch, if on one); otherwise call `findTasks(open lifecycle
  tasks)` per `reference/pm-seam.md` and ask the user which — do not guess when
  more than one matches.
- `completedStage` — the stage that just finished, passed by the stage that
  invokes the bridge. If omitted, infer it from `readTask(taskRef)` — the
  artifacts present plus the status:

  | Observed state | Last completed stage |
  |---|---|
  | Brainstorm artifact only (non-empty description, no `# Specification`) | brainstorm |
  | `# Specification`, no `# Technical plan` | spec |
  | `# Technical plan` + `# Progress`, status `Todo` | plan |
  | `# Progress` all checked, status `In Progress` | execute |
  | Status `In Review` | pr — the poller closes the card when the PR merges |
  | Status `Done` | lifecycle complete |

  If the observed state is mid-stage instead (e.g. unchecked `# Progress`
  boxes with status `In Progress`), the task's next step is *resuming that same
  stage* — offer that in the prompt instead of a successor.

## Workflow

1. **Resolve the task.** Per the Arguments rule above. If nothing resolves — no
   PM configured, or no open candidates — say so plainly and stop. That is a
   clean outcome, not an error.
2. **Determine the completed stage.** Use the `completedStage` argument, else
   infer per the Arguments table. Then look up the successor:

   | Completed stage | Next |
   |---|---|
   | brainstorm | spec |
   | spec | plan |
   | plan | execute |
   | execute | pr |
   | pr | none — lifecycle complete; the poller closes the card on merge |

3. **Handle the fixed transitions first.** These never prompt, with or without a
   horizon:
   - After **execute**: invoke `/adlc:pr` immediately, threading `taskRef`
     forward — the execute→pr hop is always automatic. Execute has already
     checkpointed every phase with the user, and the PR itself is the review
     artifact for the code; the pr stage's own gates (its unchecked-boxes
     confirmation, validation failures) remain the only stops.
   - After **pr**: do not chain — the lifecycle is complete. End the turn noting
     that the background poller `/adlc:pr` spawned will move the card to `Done`
     when the PR merges, with `/adlc:cleanup` as the fallback if the session
     ends before then.
   - A task already at `Done`: report that the lifecycle is complete and stop.
4. **Check the horizon.** The Run-to-PR horizon is active for this task only if
   the user chose **Run to PR** at one of its earlier hand-off prompts, or asked
   for it in their own words ("run it all the way to PR", "take it through the
   lifecycle", "auto mode" — any such request maps to this horizon). It is
   per-task and held in session context only — never inferred, never persisted.
   With the horizon active, skip the prompt and invoke the next stage's skill
   immediately, threading `taskRef` forward. Otherwise continue to step 5.
5. **Present the hand-off prompt.** Use the `AskUserQuestion` tool (single
   select; fall back to a plain-text question if the tool is unavailable),
   always naming the concrete next stage:
   - **Move to `<next stage>`** — invoke that stage's skill immediately,
     threading task identity per `reference/pm-seam.md` § Task Identity
     Resolution: the `taskRef` stays in session context for pre-code stages;
     code stages resolve it from the task branch. No pointer file, ever.
     When the completed stage is **plan**, label this option
     **Move to execute — continues through to PR**: execute→pr never prompts
     (step 3), so approving the plan is the last gate before the PR.
   - **Run to PR** — set the Run-to-PR horizon for this task and invoke the
     next stage immediately; every remaining hand-off chains without
     prompting. Omit this option at the plan hand-off, where it would be
     identical to Move to execute.
   - **Review first** — show what the completed stage just wrote (its Linear
     section; for `execute`, the final `# Progress` state and the branch),
     then re-present this same prompt.
   - **Stop here** — end the turn cleanly. The task is left exactly as if the
     bridge didn't exist: resumable later via the normal stage command or
     `/adlc:next`.
6. **Hand off.** Choosing **Move to next** or **Run to PR** *is* the user's
   explicit approval of the completed stage's output — the review gate that
   stage would otherwise carry itself. Invoke the next stage's skill; its own
   precondition gate still applies unchanged and will refuse if the task isn't
   actually ready.

## Boundaries

- **Never** infer the Run-to-PR horizon — it is an explicit, per-task opt-in
  (the **Run to PR** option, or the user's own words) held in session context
  for this session only. A new session starts with no horizon and prompts
  normally.
- **Never** waive a stage's preconditions or its mid-stage Ask First gates —
  the horizon and the automatic execute→pr hop skip only between-stage
  prompts; everything inside a stage (brainstorm's approval before
  `createTask`, execute's deviation Ask First, pr's unchecked-boxes
  confirmation) fires exactly as written.
- **Never** chain past `pr`, horizon or not — the lifecycle ends there; closing
  the card is the poller's job, not the bridge's.
- **Never** write to the issue from this skill — the bridge reads via the
  pm-seam only, and never invents a section name outside the canonical five.
- **Never** persist the mode or the `taskRef` to a file — session context only,
  per the pm-seam's no-pointer-file rule.

## Done when

Either the next stage's skill has been invoked with the task identity carried
forward, or the turn ended cleanly — Stop here, or lifecycle complete with the
poller watching the PR — with the task in a state indistinguishable from a run
without the bridge.
