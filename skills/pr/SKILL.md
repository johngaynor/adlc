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

## Workflow

1. **Validate first.** Run the smallest relevant set of the project's Validation
   Commands (from the root `CLAUDE.md`). If any fail, stop and report the failure —
   do **not** open a PR on red. Show the actual output.
2. **Branch if needed.** If on the default branch (`main`/`master`), create a
   feature branch first, kebab-case and prefixed by change type: `feat/`, `fix/`,
   `chore/`, `docs/`, `refactor/`.
3. **Review the diff.** Run `git status` and `git diff` (staged + unstaged). Confirm
   there are no secrets, debug output, or unrelated changes. Stage intentionally.
4. **Commit.** Write a conventional-commits message (`type: summary`, body
   explaining *why*). If the project mandates a commit trailer (check `CLAUDE.md`
   and recent `git log`), include it.
5. **Push & open.** Push the branch, then `gh pr create` against the default branch
   with a body that states: **what** changed, **why**, **how it was validated**, and
   any follow-ups. Honor `.github/pull_request_template.md` if present. Apply the
   project's PR labels if it uses a label convention.
6. **Report** the PR URL.
7. **Advance the task, if one resolves.** Call `resolveCurrentTask()` per
   [`reference/pm-seam.md`](../../reference/pm-seam.md) — it parses the current
   branch name (`<initials>/<issue-identifier>-<slug>`) to find the Linear issue.
   - If it resolves to an issue, call `setStatus(taskRef, "In Review")`. The
     branch name is also what let Linear auto-link this PR to the issue, so no
     separate linking step is needed.
   - If no PM/Linear is configured, or the current branch doesn't match the task
     branch format (an ordinary `feat/...`-style branch with no issue identifier),
     skip this step silently — no error, no prompt to configure Linear. Behave
     exactly as the generic PR opener in steps 1–6.

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
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
