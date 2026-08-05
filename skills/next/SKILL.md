---
name: next
description: Use when an ADLC lifecycle stage has just finished and the task should hand off to whatever stage comes next, or to resume an in-flight task at its next step — presents Move to next / Review first / Stop here, or chains automatically in auto mode.
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

3. **Handle the terminal row first.**
   - After **pr**: in *both* modes, do not prompt and do not chain — the
     lifecycle is complete. End the turn noting that the background poller
     `/adlc:pr` spawned will move the card to `Done` when the PR merges, with
     `/adlc:cleanup` as the fallback if the session ends before then.
   - A task already at `Done`: report that the lifecycle is complete and stop.
4. **Check the mode.** Auto mode is active only if the user explicitly opted in
   during this session ("run it in auto mode", "take it through the
   lifecycle") — never infer it. In auto mode, skip the prompt and invoke the
   next stage's skill immediately, threading `taskRef` forward. Otherwise
   continue to step 5.
5. **Present the hand-off prompt.** Use the `AskUserQuestion` tool (single
   select; fall back to a plain-text question if the tool is unavailable),
   always naming the concrete next stage:
   - **Move to `<next stage>`** — invoke that stage's skill immediately,
     threading task identity per `reference/pm-seam.md` § Task Identity
     Resolution: the `taskRef` stays in session context for pre-code stages;
     code stages resolve it from the task branch. No pointer file, ever.
   - **Review first** — show what the completed stage just wrote (its Linear
     section; for `execute`, the final `# Progress` state and the branch),
     then re-present this same prompt.
   - **Stop here** — end the turn cleanly. The task is left exactly as if the
     bridge didn't exist: resumable later via the normal stage command or
     `/adlc:next`.
6. **Hand off.** Choosing **Move to next** *is* the user's explicit approval of
   the completed stage's output — the review gate that stage would otherwise
   carry itself. Invoke the next stage's skill; its own precondition gate still
   applies unchanged and will refuse if the task isn't actually ready.

## Boundaries

- **Never** infer auto mode — it is an explicit, user-stated opt-in held in
  session context for this session only.
- **Never** waive a stage's preconditions or its mid-stage Ask First gates —
  auto mode skips only this between-stage prompt; everything inside a stage
  (brainstorm's approval before `createTask`, execute's deviation Ask First,
  pr's unchecked-boxes confirmation) fires exactly as written.
- **Never** chain past `pr`, in either mode — the lifecycle ends there; closing
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
