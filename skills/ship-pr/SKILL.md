---
name: ship-pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". Runs validation, commits on a fresh branch, pushes, and opens a PR with a structured description.
---

# Ship a PR

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description. This skill commits and
pushes — it runs only when the user has explicitly asked to open a PR.

## Workflow

1. **Validate first.** Run the smallest relevant set of the project's Validation
   Commands (from the root `CLAUDE.md`). If any fail, stop and report the failure —
   do **not** open a PR on red. Show the actual output.
2. **Branch if needed.** First detect a managed worktree: `CONDUCTOR_WORKSPACE_NAME`
   is set in the environment, or `git rev-parse --git-common-dir` resolves outside
   this checkout's own `.git`. In a managed worktree the current branch *is* the
   feature branch — use it as-is; do **not** create a second branch or rename it.
   Otherwise, if on the default branch (`main`/`master`), create a feature branch
   first, kebab-case and prefixed by change type: `feat/`, `fix/`, `chore/`,
   `docs/`, `refactor/`. The naming convention applies only to branches this skill
   creates.
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

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
- **Never** commit secrets, credentials, or tokens.
- **Never** open a PR with failing validation, or claim validation passed without
  running it.

## Done when

The PR is open, validation passed with evidence, and the URL is reported.
