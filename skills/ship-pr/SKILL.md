---
name: ship-pr
description: Use when work is complete and the user wants to open a pull request — "ship it", "open a PR", "put up a PR". Runs validation, commits on a fresh branch, pushes, and opens a PR with a structured description.
---

# Ship a PR

> ⚠️ **STUB — Stream C.** This skill is scaffolded but not yet implemented. Fill in
> the Workflow below following [`CONVENTIONS.md`](../../CONVENTIONS.md) and
> [`METHODOLOGY.md`](../../METHODOLOGY.md). Remove this banner when done.

Take completed work and ship it as a reviewable PR with the project's discipline:
validation green, a clean branch, a clear description.

## Build contract (what this skill must do)

1. Run the project's Validation Commands (from the root `CLAUDE.md`) and refuse to
   proceed if any fail — report the failure instead.
2. If on the default branch, create a feature branch first (kebab-case, prefixed by
   change type, e.g. `feat/`, `fix/`, `chore/`).
3. Stage and commit with a conventional-commits message. End commit messages with
   the project's required trailer if one exists.
4. Push and open the PR with `gh pr create`, using a description that states: what
   changed, why, how it was validated, and any follow-ups. Honor a
   `.github/pull_request_template.md` if present.
5. Report the PR URL.

**Boundaries:** Ask First before force-pushing or merging. Never commit secrets.
Never open a PR with failing validation. Commit/push only as part of this
explicitly-requested flow.

**Done when:** the PR is open, validation passed, and the URL is reported.
