---
name: archive
description: Use when an ADLC task's PR has merged and the work is done — the final lifecycle stage. Writes a curated summary to the repo and closes the Linear issue.
---

# Archive

Close out a finished task: leave a curated, durable summary in the repo and close
the Linear issue — the sixth and final stage of the ADLC lifecycle
(`brainstorm → spec → plan → execute → pr → archive`, see
[`METHODOLOGY.md`](../../METHODOLOGY.md)). The issue already holds the full
`## Specification` and `## Plan` for anyone who wants the detail — the repo only
needs what shipped, why, and how, so it stays a light record rather than a
duplicate of Linear.

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
3. **Draft the curated summary.** Read `## Idea`, `## Specification`, `## Plan`,
   and `## Progress` for context, then write a short new document — **not** a
   copy of any of them — covering:
   - **What shipped** — the concrete, user-visible outcome.
   - **Why** — the problem it solved, in a sentence or two.
   - **How** — the approach in brief, not a phase-by-phase replay of `## Plan`.
   - **Notable deviations from the plan** — anything flagged as a deviation
     during `/adlc:execute` (see its Boundaries); write "None" if there weren't
     any.
4. **Render the file.** Copy
   `${CLAUDE_PLUGIN_ROOT}/templates/archive-summary.md.template` (reproduce its
   sections inline if unresolvable) into `docs/adlc/{YYYY-MM-DD}-{kebab-slug}.md`
   — date from `date +%F`, slug derived from the issue title — creating
   `docs/adlc/` if it doesn't exist. Fill in the title, date, Linear issue link,
   PR link(s), and the four drafted sections.
5. **Commit the summary** on the task branch with a message naming the task.
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
- **Never** copy the full `## Specification` or `## Plan` verbatim into the repo
  summary — curate it down to what/why/how and deviations; the issue remains the
  detailed record.
- **Never** run this stage against a task whose PR isn't merged yet.
- **Never** invent a Linear section name outside the five canonical ones in
  `reference/pm-seam.md`.

## Done when

`docs/adlc/{YYYY-MM-DD}-{kebab-slug}.md` is committed with a curated summary, and
the issue's status is `Done`.
