---
name: brainstorm
description: Use when starting a new piece of work from a rough idea — the user wants to kick off an ADLC task. Runs a collaborative dialogue to sharpen the idea, then creates the Linear issue that will hold the whole task.
---

# Brainstorm

Sharpen a rough idea into a crisp problem statement, then create the task's Linear
issue — the first stage of the ADLC lifecycle (`brainstorm → spec → plan → execute →
pr → archive`, see [`METHODOLOGY.md`](../../METHODOLOGY.md)). This is the only
lifecycle stage that runs before a Linear issue exists; every later stage assumes one.

## Workflow

1. **Dialogue with the user.** Ask one question at a time to clarify purpose,
   constraints, and success criteria. Do not invent scope the user hasn't stated —
   if something is unclear, ask rather than assume.
2. **Draft the idea.** Write a concise title and an `## Idea` markdown block covering
   the problem, why now, and rough success criteria.
3. **Confirm with the user.** Present the title and `## Idea` draft and get explicit
   approval. This gate happens **before** any issue is created — never create the
   issue on a draft the user hasn't seen.
4. **Create the task.** Call `createTask(title, ideaMarkdown)` per
   [`reference/pm-seam.md`](../../reference/pm-seam.md). This creates the Linear
   issue with the approved title and `## Idea` body, and sets its status to
   `Backlog` — no separate `setStatus` call is needed; `createTask` sets it per the
   seam's Status Mapping.
5. **Report and hand off.** Give the user the issue URL and hold its `taskRef` in
   session context — there is no shared pointer file (see `reference/pm-seam.md`
   § Task Identity Resolution). Then invoke the workflow bridge
   ([`/adlc:next`](../next/SKILL.md)) with `completedStage: brainstorm` and the
   `taskRef` — the bridge owns the lifecycle order and what comes next.

## Boundaries

- **Never** create the Linear issue before the user has approved the idea draft.
- **Never** write code, create a branch, or touch the task's worktree here — that
  starts at `/adlc:execute`.
- **Ask First** before broadening scope beyond what the user described.

## Done when

A `Backlog` Linear issue exists with an `## Idea` section, and the user has its URL.
