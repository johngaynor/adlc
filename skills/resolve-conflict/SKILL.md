---
name: resolve-conflict
description: Use when a PR has merge conflicts with the default branch — GitHub shows "this branch has conflicts", `gh pr view` reports `mergeable: CONFLICTING`, or the pr skill's post-merge poller finds a conflicted task PR. Merges the default branch into the PR branch in an isolated worktree, resolves only trivially-resolvable conflicts, re-validates, and pushes — escalating anything non-trivial instead of guessing.
---

# Resolve Conflict

Bring a conflicted PR branch back to mergeable by merging the default branch into
it and resolving only conflicts that are trivial beyond doubt. This is a
**utility skill, not a lifecycle stage** (see
[`METHODOLOGY.md`](../../METHODOLOGY.md)) — the same posture as `/adlc:cleanup`:
it can run at any time, writes no card sections, labels, or statuses, and does
not invoke `/adlc:next`. It touches git and `gh` only. Safety comes from three
hard rules: merge, never rebase (so a plain push always suffices and force-push
never enters the picture); resolve all-or-nothing (one doubtful hunk aborts the
whole merge); and validation gates the push.

## Arguments

- `pr` (optional) — the PR URL, number, or head branch name to resolve. If
  omitted, resolve the open PR of the current branch (`gh pr view --json url`);
  if that finds nothing, ask the user which PR to work on — do not guess.

## Modes

The procedure is identical in both modes; only the escalation path differs.

- **Background mode** — invoked by the post-merge poller `/adlc:pr` spawns
  (see [`skills/pr/SKILL.md`](../pr/SKILL.md) step 9). Fully unattended: trivial
  conflicts only. Anything non-trivial → abort, report, touch nothing.
- **Standalone mode** — invoked directly (`/adlc:resolve-conflict`), a user
  present. Runs the same trivial-first procedure; when a non-trivial hunk stops
  it, it may then present the conflicted hunks and both sides' intent and work
  through them **with** the user (Ask First per hunk) instead of stopping at a
  report.

## Workflow

1. **Confirm the conflict.** Run
   `gh pr view <pr> --json state,mergeable,mergeStateStatus,headRefName,baseRefName`.
   Proceed only on `mergeable: CONFLICTING` with state `OPEN`. Treat
   `mergeable: UNKNOWN` as "GitHub is still computing" — it is not a conflict;
   in background mode check again next poll, in standalone mode wait briefly and
   re-check. A merged or closed PR ends the skill immediately.
2. **Set up the workspace.** Never work in the main checkout (METHODOLOGY § 5).
   Run `git fetch origin` first. Reuse the task's existing worktree **only if**
   it is on the PR's head branch, `git -C <worktree> status --porcelain` is
   empty, and the branch is in sync with the remote head. Otherwise create a
   disposable one — `git worktree add .worktrees/resolve-<branch-slug>
   <head-branch>` — and remove it in step 7.
3. **Merge, never rebase.** In the workspace:
   `git merge origin/<base-branch>`. A merge commit is append-only history, so
   step 6 needs only a plain push; rebasing would rewrite pushed commits and
   require a force-push, which this skill never does in any mode.
4. **Apply the triviality test.** Inspect every conflicted hunk
   (`git diff --name-only --diff-filter=U`, then the conflict markers). Each
   hunk must fall into one of exactly three categories:
   - **Regenerable** — lockfiles or generated files with a known generator:
     resolve by re-running the generator, never by hand-editing the file.
   - **Union-safe** — both sides appended or inserted independent content
     (adjacent-line edits; append-style files like `lessons.md` or changelogs):
     resolve by keeping both sides.
   - **Equivalent** — the sides differ only in whitespace or formatting, or are
     textually identical: resolve to either.

   Anything that requires choosing one side's logic over the other's, editing
   either side's content, or writing new code to reconcile them is
   **non-trivial** — and one non-trivial hunk fails the whole merge: run
   `git merge --abort`, leave the branch and PR untouched, and escalate per the
   mode (background: report and stop; standalone: offer to work the hunks with
   the user). No partial resolutions, ever.
5. **Validate before pushing.** Run the smallest relevant set of the consuming
   repo's Validation Commands (root `CLAUDE.md`, per
   [`METHODOLOGY.md`](../../METHODOLOGY.md) § 3) in the workspace. If anything
   fails, do **not** push — reset the merge (`git reset --hard
   origin/<head-branch>`) and escalate exactly as if the conflict were
   non-trivial, including the failing output in the report.
6. **Push.** Commit the merge if needed and run a plain `git push` — never
   `--force` or `--force-with-lease`. If the push is rejected non-fast-forward
   (someone pushed to the branch mid-resolution), discard the local merge and
   end this attempt — in background mode the next poll retries against the new
   head; in standalone mode re-run from step 1.
7. **Clean up and report.** Remove any worktree created in step 2
   (`git worktree remove ...`). Report what happened: conflicted files, the
   category each hunk resolved under (or the hunks that failed the test),
   validation evidence, and whether the branch was pushed.

## Boundaries

- **Never** rebase or force-push — in either mode, for any reason.
- **Never** resolve partially: one non-trivial hunk aborts the entire merge.
- **Never** hand-edit content to reconcile the two sides — that is a human
  decision; the three categories are the whole authority of this skill.
- **Never** work in the main checkout, or in a dirty or out-of-sync worktree.
- **Never** write card sections, labels, or statuses — this is a utility
  skill outside the lifecycle; the card is not this skill's concern.
- **Ask First** (standalone mode only) before resolving any non-trivial hunk
  with the user — present both sides' intent and get an explicit choice per
  hunk. In background mode there is no one to ask: escalation is report-only.

## Done when

The PR reports `mergeable: MERGEABLE` with the merge commit pushed and
validation evidence in the report — or the merge was cleanly aborted, the
branch and PR are byte-for-byte untouched, and the report says exactly which
hunks failed the triviality test and why.
