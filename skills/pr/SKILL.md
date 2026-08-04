---
name: pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". This is the ADLC `pr` lifecycle stage: runs validation, commits, pushes, opens a PR with a structured description, and advances the task to `In Review`.
---

# Open a PR

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description — the fifth stage of the ADLC
lifecycle (`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). This skill commits and pushes — it runs
only when the user has explicitly asked to open a PR. Like every lifecycle stage it
requires Linear (see [`reference/pm-seam.md`](../../reference/pm-seam.md)
§ Preconditions Contract): it resolves the task from the current branch and
advances it to `In Review` once the PR is open.

Run this stage from the task worktree `execute` created — its branch is the task
pointer `resolveCurrentTask` parses. If the current checkout isn't on the task
branch, `cd` into that worktree first.

## Workflow

1. **Validate first.** Run the smallest relevant set of the Validation Commands
   from the consuming repo's root `CLAUDE.md`. If any fail, stop and report the
   failure — do **not** open a PR on red. Show the actual output.
2. **Resolve the task — a precondition.** Call `resolveCurrentTask()` per
   [`reference/pm-seam.md`](../../reference/pm-seam.md) — it parses the current
   branch name (`<initials>/<issue-identifier>-<slug>`) to find the Linear issue.
   - If Linear isn't configured (no `pm` block in `.adlc/config.json`), refuse
     with the seam's standard message: "Linear isn't configured for this repo —
     run `/adlc:init` to connect it."
   - If the current branch doesn't parse as a task branch, refuse and explain:
     this isn't a task branch, and the lifecycle's `/adlc:execute` is what
     creates one. Never open a PR from an unresolved branch.
   - Once resolved, call `readTask(taskRef)` and check the `## Progress`
     checklist. If any box is still unchecked, warn the user that not every
     planned phase is done and ask for confirmation before proceeding — do not
     hard-block; shipping partial work as a PR can be intentional.
3. **Review the diff.** Run `git status` and `git diff` (staged + unstaged). Confirm
   there are no secrets, debug output, or unrelated changes. Stage intentionally.
4. **Commit.** Write a conventional-commits message (`type: summary`, body
   explaining *why*). If the project mandates a commit trailer (check `CLAUDE.md`
   and recent `git log`), include it.
5. **Push & open.** Push the branch, then `gh pr create` against the default branch
   with a body that states: **what** changed, **why**, **how it was validated**, and
   any follow-ups. Honor `.github/pull_request_template.md` if present. Apply the
   project's PR labels if it uses a label convention. The task branch name is what
   lets Linear auto-link the PR to the issue — no separate linking step is needed.
6. **Report** the PR URL. Note that the task worktree is now disposable — the
   branch lives on the remote.
7. **Advance the task.** Reuse the `taskRef` resolved in step 2 and call
   `setStatus(taskRef, "In Review")`.

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
- **Ask First** before opening the PR when the task has unchecked `## Progress`
  boxes — warn and get confirmation, never hard-block.
- **Never** commit secrets, credentials, or tokens.
- **Never** open a PR with failing validation, or claim validation passed without
  running it.
- **Never** let a failing `In Review` status update retroactively fail an
  already-opened PR — report the error and tell the user to set the status by
  hand; the PR stands.

## Done when

The PR is open, validation passed with evidence, the URL is reported, and the
task's status is `In Review`.
