---
name: archive
description: Use when an ADLC task's PR has merged and the work is done — the final lifecycle stage. Exports the final spec and plan to .ai/specs/implemented/ and closes the Linear issue.
---

# Archive

Close out a finished task: export its final specification and plan from the Linear
issue into the repo as the durable record, optionally wire that record into the
Task Router, and close the issue — the sixth and final stage of the ADLC lifecycle
(`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). The issue is the working surface while
the task is in flight; the export makes the repo the greppable, in-repo record
future agent sessions are routed into — no retro or separately-authored summary
is written.

Run this stage from the task worktree `execute` created — its branch is the
task pointer `resolveCurrentTask` parses. If the current checkout isn't on the
task branch, `cd` into that worktree first.

## Arguments

- `taskRef` (optional) — the Linear issue URL or identifier to archive. If
  omitted, call `resolveCurrentTask()` per
  [`reference/pm-seam.md`](../../reference/pm-seam.md) — it parses the current
  git branch (`<initials>/<issue-identifier>-<slug>`) to find the issue.

## Workflow

1. **Resolve the task.** Determine `taskRef` per the Arguments rule above. Refuse
   if it doesn't resolve — never guess which issue this is.
2. **Read and gate on the precondition.** Call `readTask(taskRef)`, then confirm
   the task's PR is merged (check the PR opened by `/adlc:pr`, or ask the user
   for its URL/state if it isn't already known). If it is not merged, refuse and
   tell the user to finish `/adlc:pr` and get it merged first.
3. **Assemble the export.** From `readTask`, take the final `## Specification`
   and `## Plan` sections **verbatim** — do not rewrite, trim, or summarize
   them. Assemble `## Notable deviations` from the deviations `/adlc:execute`
   flagged in commit messages and checkpoint reports (see its Boundaries);
   write "None." if there weren't any.
4. **Render the file.** Target path is
   `.ai/specs/implemented/{YYYY-MM-DD}-{kebab-slug}.md` — date from `date +%F`,
   slug derived from the issue title. Two guards before writing:
   - **Idempotency** (see `CONVENTIONS.md`): if the file already exists, show
     its current contents to the user and ask before overwriting — never
     clobber it silently.
   - **Gitignore:** run `git check-ignore -q` on the target path. If it is
     ignored (first run in this repo), make the path trackable with the
     `*`-pattern ladder — a `!` re-include only works when parent directories
     use `*` patterns, not directory ignores:
     ```
     .ai/*
     !.ai/specs/
     .ai/specs/*
     !.ai/specs/implemented/
     ```
     (adapt to the repo's existing `.ai` entries; re-run `git check-ignore` to
     confirm before proceeding).
   Once clear to write, copy
   `${CLAUDE_PLUGIN_ROOT}/templates/spec-export.md.template` (reproduce its
   sections inline if unresolvable) into that path, creating the directory if
   needed. Fill in the title, date, Linear issue link, merged PR link(s), and
   the three sections from step 3.
5. **Judge the Task Router.** Did this task introduce a pattern, subsystem, or
   contract future agents will touch again? If yes, draft an add-or-update to
   one Task Router row in the consuming repo's root `CLAUDE.md`, linking the
   exported spec (task → guide, per `METHODOLOGY.md` § 2). If it's a one-off,
   skip — the export alone is enough; a router full of one-offs stops routing.
6. **Confirm with the user.** Show the rendered export, the router judgment
   (the proposed row, or why none), and any gitignore edit — and get explicit
   approval. This is the Ask First gate for everything that follows: the
   archive PR and closing the issue.
7. **Land the export via an auto-merged docs-only PR.** By the time this stage
   runs, the task's *code* PR (from `/adlc:pr`) is already merged, so the export
   needs its own way onto the base branch. Commit it on a short docs branch (e.g.
   `<initials>/<issue-identifier>-archive`), push, and open a PR against the base.
   This PR carries **at most three files** — the exported spec, the `CLAUDE.md`
   router row, and (first run only) the gitignore ladder; all harness/docs, no
   runtime code or behavior change — so it needs no code review or QA:
   auto-approve it and enable auto-merge (`gh pr merge --auto --squash`, plus
   the repo's auto-merge / skip-review label if it uses one). If the repo allows
   direct commits to the base branch, commit directly and skip the PR.
8. **Optionally record the outcome in Linear.** Call `writeSection(taskRef,
   "Outcome", prLinks)` per `reference/pm-seam.md` with the merged PR link(s), so
   the issue itself also shows how the task ended.
9. **Close the task.** Call `closeTask(taskRef)` — sets status `Done`.

## Boundaries

- **Ask First** at step 6 — never open the archive PR or call `closeTask`
  without the user's explicit go-ahead on the export, the router judgment, and
  any gitignore edit.
- **Never** overwrite an existing
  `.ai/specs/implemented/{YYYY-MM-DD}-{kebab-slug}.md` without showing its
  contents and getting the user's go-ahead first — this stage runs on the same
  date/slug if re-invoked, so it must be safe to run twice.
- **The archive PR carries at most three files** — the exported spec, the
  `CLAUDE.md` Task Router row, and (first run) the gitignore ladder — and may be
  auto-approved and auto-merged; none of them changes runtime behavior. This is
  the deliberate exception to the review discipline that governs the *code* PR
  from `/adlc:pr`; **never** fold code or behavior changes into an archive PR to
  ride its auto-merge.
- **Never** summarize, trim, or rewrite the `## Specification` or `## Plan` in
  the export — they land verbatim; the export is the record, not a digest.
- **Never** add a Task Router row for a one-off task — route only durable
  patterns, subsystems, or contracts.
- **Never** run this stage against a task whose PR isn't merged yet.
- **Never** invent a Linear section name outside the five canonical ones in
  `reference/pm-seam.md`.

## Done when

`.ai/specs/implemented/{YYYY-MM-DD}-{kebab-slug}.md` is committed with the
task's final spec, plan, and deviations (plus a Task Router row when the task
earned one), and the issue's status is `Done`.
