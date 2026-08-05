---
name: archive
description: Use when an ADLC task's PR has merged and the work is done — the final lifecycle stage. Writes a curated summary to the repo and closes the Linear issue.
---

# Archive

Close out a finished task: leave a curated, durable summary in the repo and close
the Linear issue — the sixth and final stage of the ADLC lifecycle
(`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). The issue already holds the full
`# Specification` and `## Plan` for anyone who wants the detail — the repo only
needs what shipped, why, and how, so it stays a light record rather than a
duplicate of Linear.

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
3. **Draft the curated summary.** Read the brainstorm artifact, `# Specification`, `## Plan`,
   and `## Progress` for context, then write a short new document — **not** a
   copy of any of them — covering:
   - **What shipped** — the concrete, user-visible outcome.
   - **Why** — the problem it solved, in a sentence or two.
   - **How** — the approach in brief, not a phase-by-phase replay of `## Plan`.
   - **Notable deviations from the plan** — anything flagged as a deviation
     during `/adlc:execute` (see its Boundaries); write "None" if there weren't
     any.
4. **Render the file.** Target path is `docs/adlc/{YYYY-MM-DD}-{kebab-slug}.md` —
   date from `date +%F`, slug derived from the issue title. This stage must be
   safe to run twice (see `CONVENTIONS.md`): before writing, check whether that
   file already exists. If it does, show its current contents to the user and
   ask before overwriting — never clobber it silently. Once clear to write (the
   file doesn't exist, or the user confirmed the overwrite), copy
   `${CLAUDE_PLUGIN_ROOT}/templates/archive-summary.md.template` (reproduce its
   sections inline if unresolvable) into that path, creating `docs/adlc/` if it
   doesn't exist. Fill in the title, date, Linear issue link, PR link(s), and
   the four drafted sections.
5. **Land the summary via an auto-merged docs-only PR.** By the time this stage
   runs, the task's *code* PR (from `/adlc:pr`) is already merged, so the summary
   needs its own way onto the base branch. Commit it on a short docs branch (e.g.
   `<initials>/<issue-identifier>-archive`), push, and open a PR against the base.
   This PR is **notes only** — a single documentation file, no code, no behavior
   change — so it needs no code review or QA: auto-approve it and enable
   auto-merge (`gh pr merge --auto --squash`, plus the repo's auto-merge /
   skip-review label if it uses one). If the repo allows direct commits to the
   base branch, commit the summary there directly and skip the PR.
6. **Optionally record the outcome in Linear.** Call `writeSection(taskRef,
   "Outcome", prLinks)` per `reference/pm-seam.md` with the merged PR link(s), so
   the issue itself also shows how the task ended.
7. **Confirm with the user before closing.** Show the rendered summary and get
   explicit approval — this is the Ask First gate on closing the issue.
8. **Close the task.** Once approved, call `closeTask(taskRef)` — sets status
   `Done`.

## Boundaries

- **Ask First** before calling `closeTask` — never close the issue without the
  user's explicit go-ahead on the drafted summary.
- **Never** overwrite an existing `docs/adlc/{YYYY-MM-DD}-{kebab-slug}.md`
  without showing its contents and getting the user's go-ahead first — this
  stage runs on the same date/slug if re-invoked, so it must be safe to run
  twice.
- **The archive summary PR is notes only** and may be auto-approved and
  auto-merged — it carries one documentation file and nothing else. This is the
  deliberate exception to the review discipline that governs the *code* PR from
  `/adlc:pr`; **never** fold code or behavior changes into an archive PR to ride
  its auto-merge.
- **Never** copy the full `# Specification` or `## Plan` verbatim into the repo
  summary — curate it down to what/why/how and deviations; the issue remains the
  detailed record.
- **Never** run this stage against a task whose PR isn't merged yet.
- **Never** invent an artifact outside the canonical layout in
  `reference/pm-seam.md`.

## Done when

`docs/adlc/{YYYY-MM-DD}-{kebab-slug}.md` is committed with a curated summary, and
the issue's status is `Done`.
