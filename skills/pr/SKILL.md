---
name: pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". This is the ADLC `pr` lifecycle stage: runs validation, commits on a fresh branch, pushes, opens a PR with a structured description, and — when a Linear task resolves — advances it to `In Review` and spawns a background poller that moves it to `Done` once the PR merges.
---

# Open a PR

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description — the fifth and final stage
of the ADLC lifecycle (`brainstorm → spec → plan → execute → pr`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). This skill commits and pushes — it runs
only when the user has explicitly asked to open a PR. It works standalone with no PM
configured, exactly as it did before the lifecycle existed; when a Linear task
resolves from the current branch, it also advances that task to `In Review` and
leaves a background poller behind to move it to `Done` when the PR merges.

When a task is in play, run this stage from the task worktree `execute`
created — its branch is the task pointer `resolveCurrentTask` parses. If the
current checkout isn't on the task branch, `cd` into that worktree first.

## Workflow

1. **Validate first.** Run the smallest relevant set of the Validation Commands
   from the consuming repo's root `CLAUDE.md`. If any fail, stop and report the
   failure — do **not** open a PR on red. Show the actual output.
2. **Check task progress, if one resolves.** Call `resolveCurrentTask()` per
   [`reference/pm-seam.md`](../../reference/pm-seam.md) — it parses the current
   branch name (`<initials>/<issue-identifier>-<slug>`) to find the Linear issue.
   This is best-effort, not a precondition: `pr` also works with no PM/Linear
   configured at all.
   - If it resolves to an issue, call `readTask(taskRef)` and check the
     `# Progress` checklist. If any box is still unchecked, warn the user that
     not every planned phase is done and ask for confirmation before proceeding
     — do not hard-block; shipping partial work as a PR can be intentional.
   - If no task resolves (no PM/Linear configured, or the current branch doesn't
     match the task branch format), proceed with no further action here — this
     stage still works as a plain PR opener.
3. **Branch if needed.** Get on a proper feature branch (kebab-case, prefixed by
   change type: `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`) per
   METHODOLOGY.md idea 5:
   - On a machine-generated branch (e.g. `worktree-*`): rename it
     (`git branch -m <new-name>`) before pushing.
   - On the default branch (`main`/`master`): create the feature branch first.
   - On a sensibly-named branch already (a task branch or platform-managed
     name): keep it.
4. **Review the diff.** Run `git status` and `git diff` (staged + unstaged). Confirm
   there are no secrets, debug output, or unrelated changes. Stage intentionally.
5. **Commit.** Write a conventional-commits message (`type: summary`, body
   explaining *why*). If the project mandates a commit trailer (check `CLAUDE.md`
   and recent `git log`), include it.
6. **Push & open.** Push the branch, then `gh pr create` against the default branch
   with a body that states: **what** changed, **why**, **how it was validated**, and
   any follow-ups. Honor `.github/pull_request_template.md` if present. Apply the
   project's PR labels if it uses a label convention.
7. **Report** the PR URL. If working in an isolated workspace (worktree), note
   that it is now disposable — the branch lives on the remote.
8. **Advance the task, if one resolved.** Reuse the `taskRef` resolved in step 2.
   - If it resolved to an issue, call `setStatus(taskRef, "In Review")`. The
     branch name is also what let Linear auto-link this PR to the issue, so no
     separate linking step is needed.
   - If no PM/Linear is configured, or the current branch doesn't match the task
     branch format (an ordinary `feat/...`-style branch with no issue identifier),
     skip this step silently — no error, no prompt to configure Linear. Behave
     exactly as the generic PR opener in the rest of this workflow.
9. **Spawn the post-merge poller, if a task resolved.** Only when step 2
   resolved a Linear issue — with no task there is nothing to close, so skip
   this step silently. Spawn a **background subagent** (the harness's background
   agent facility) whose sole job is:
   - Hold an **active polling loop for its entire lifetime**: check, wait
     in-process, check again. The poller must **never** arm a watcher, monitor,
     or scheduled wake-up and then exit, expecting an event to resume it — that
     pattern has already failed in practice (PHY-168: the agent armed a
     4-minute monitor and exited; the monitor never fired, and the card sat in
     `In Review` for 20+ minutes after the merge until it was closed by hand).
     An agent that cannot hold a live loop in its harness must say so in its
     spawn report instead of pretending to poll — the card then falls to the
     `/adlc:cleanup` safety net knowingly, not silently.
   - Poll the PR's merge state at a modest interval (a few minutes between
     checks): `gh pr view <pr-url> --json state,mergedAt,mergeable,mergeStateStatus`.
   - When the PR reports merged, call `closeTask(taskRef)` per
     [`reference/pm-seam.md`](../../reference/pm-seam.md) — the issue moves to
     `Done` — and report that it did so.
   - When the PR reports `mergeable: CONFLICTING`, run the
     [`resolve-conflict`](../resolve-conflict/SKILL.md) procedure in its
     background mode: trivial conflicts are resolved, validated, and plain-pushed;
     anything non-trivial is reported and left untouched. `mergeable: UNKNOWN`
     means GitHub is still computing — not a conflict; check again next poll.
     Cap resolution at **2 attempts per PR** — a conflict recurring after a
     successful resolution means the default branch keeps moving in the same
     area and a human should look. Report every action taken (resolved and
     pushed, escalated, or capped out).
   - If the PR is closed without merging, or the poll budget runs out (stop
     after roughly a day of polling), report and leave the card untouched.
   - The poller writes **nothing else** — no sections, no labels, no
     worktree/branch removal. Its only write outside the `Done` status is the
     merge-commit push made through the `resolve-conflict` procedure above.

   Known limitation: the poller lives only as long as this session. If the
   session ends before the PR merges, the card stays `In Review` until
   `/adlc:cleanup` — the safety net — catches it.
10. **Hand off.** Invoke the workflow bridge ([`/adlc:next`](../next/SKILL.md))
    with `completedStage: pr` and the `taskRef` (when one resolved; with no
    resolved task the bridge degrades silently, matching this stage's PM-optional
    behavior). `pr` is the final lifecycle stage, so the bridge ends the turn —
    noting that the poller will close the card on merge.

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
- **Ask First** before opening the PR when a resolved task has unchecked
  `# Progress` boxes — warn and get confirmation, never hard-block.
- **Never** commit secrets, credentials, or tokens.
- **Never** open a PR with failing validation, or claim validation passed without
  running it.
- **Never** let the `In Review` status update block or fail PR creation — task
  resolution is best-effort and always secondary to the PR itself, and its absence
  (no PM configured, or an unmapped branch) is not an error.
- **Never** let the poller write anything beyond the `Done` status and the
  merge-commit push made through the
  [`resolve-conflict`](../resolve-conflict/SKILL.md) procedure — no sections,
  no labels, no worktree or branch removal.
- **Never** let the poller force-push or resolve a conflict outside
  `resolve-conflict`'s triviality test — non-trivial conflicts are reported,
  never guessed at.
- **Never** let the poller arm-and-exit — an exited agent waiting to be resumed
  by a watcher or monitor is not a poller (PHY-168 proved the resume never
  comes). It loops actively for its whole lifetime, or reports that it can't.
- **Never** let a poller spawn failure block or fail PR creation — report it and
  point at `/adlc:cleanup` as the fallback.

## Done when

The PR is open, validation passed with evidence, and the URL is reported. When
`resolveCurrentTask()` resolved a Linear issue from the branch, its status is now
`In Review` and the background poller is watching the PR — moving it to `Done` on
merge, and clearing trivial merge conflicts along the way via
[`resolve-conflict`](../resolve-conflict/SKILL.md); when it didn't (no PM
configured, or a non-task branch), the PR alone is the completed artifact.
