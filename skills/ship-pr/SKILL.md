---
name: ship-pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". Runs validation, commits on the feature branch (creating it if needed), pushes, and opens a PR with a structured description.
---

# Ship a PR

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description. This skill commits and
pushes — it runs only when the user has explicitly asked to open a PR.

## Workflow

1. Run the project's Validation Commands (from the root `CLAUDE.md`) and refuse to
   proceed if any fail — report the failure instead, showing the actual output.
2. Get on a proper feature branch (kebab-case, prefixed by change type, e.g.
   `feat/`, `fix/`, `chore/`) per METHODOLOGY.md idea 5. A managed worktree is
   detected by `CONDUCTOR_WORKSPACE_NAME` being set in the environment, or by
   `git rev-parse --git-dir` and `git rev-parse --git-common-dir` printing
   different paths (they differ exactly in a linked worktree). Then:
   - On a machine-generated branch (e.g. `worktree-*`): rename it
     (`git branch -m <new-name>`) before pushing.
   - On a sensibly-named branch already (e.g. platform-managed, such as a
     Conductor workspace): keep it — do **not** create a second branch or
     rename it.
   - On the default branch (`main`/`master`) — even inside a linked worktree
     (e.g. `git worktree add ../main main`): create the feature branch first.
   The naming convention applies only to branches this skill creates or renames.
3. Stage and commit with a conventional-commits message. End commit messages with
   the project's required trailer if one exists.
4. Push and open the PR with `gh pr create`, using a description that states: what
   changed, why, how it was validated, and any follow-ups. Honor a
   `.github/pull_request_template.md` if present.
5. Report the PR URL. If working in an isolated workspace (worktree), note that
   it is now disposable — the branch lives on the remote.

## Boundaries

- **Ask First** before force-pushing, merging, or targeting a non-default base.
- **Never** commit secrets, credentials, or tokens.
- **Never** open a PR with failing validation, or claim validation passed without
  running it.

## Done when

The PR is open, validation passed with evidence, and the URL is reported.
