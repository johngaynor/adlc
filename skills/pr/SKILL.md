---
name: pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". This is the ADLC `pr` lifecycle stage: runs validation, commits on a fresh branch, pushes, opens a PR with a structured description, and advances the task to `In Review` when a Linear task resolves.
---

# Open a PR

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description — the fifth stage of the ADLC
lifecycle (`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). This skill commits and pushes — it runs
only when the user has explicitly asked to open a PR. It works standalone with no PM
configured, exactly as it did before the lifecycle existed; when a Linear task
resolves from the current branch, it also advances that task to `In Review`.

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
     `## Progress` checklist. If any box is still unchecked, warn the user that
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

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
- **Ask First** before opening the PR when a resolved task has unchecked
  `## Progress` boxes — warn and get confirmation, never hard-block.
- **Never** commit secrets, credentials, or tokens.
- **Never** open a PR with failing validation, or claim validation passed without
  running it.
- **Never** let the `In Review` status update block or fail PR creation — task
  resolution is best-effort and always secondary to the PR itself, and its absence
  (no PM configured, or an unmapped branch) is not an error.

## Done when

The PR is open, validation passed with evidence, and the URL is reported. When
`resolveCurrentTask()` resolved a Linear issue from the branch, its status is now
`In Review`; when it didn't (no PM configured, or a non-task branch), the PR alone
is the completed artifact.
