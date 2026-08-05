---
name: cleanup
description: Use when merged work has left debris behind — stale worktrees, dead local/remote branches, or PM cards whose PR merged but never reached Done. Sweeps both automatically and reports what was removed and what was skipped and why.
---

# Cleanup

Sweep the debris that accumulates after PRs merge: task worktrees (and their
local/remote branches) that lingered past their merge, and PM issues whose
PR landed but whose status never advanced to `Done`. This is a **utility skill,
not a lifecycle stage** (see [`METHODOLOGY.md`](../../METHODOLOGY.md)): it can
run at any time, writes no card sections or labels, and does not invoke
`/adlc:next`. It runs fully automatically — no per-item confirmation prompts.
Safety comes from conservative criteria instead: only worktrees that are
**merged AND clean** are ever touched, so unmerged or dirty work (including
parallel in-flight sessions) is never at risk.

## Workflow

1. **Sync state.** Run `git fetch origin` (substitute the repo's actual remote),
   then `git worktree prune` to drop stale registrations, then enumerate
   `git worktree list --porcelain`.

2. **Worktree sweep.** Candidates are **linked worktrees only** — skip the main
   checkout, the worktree the current session is running in, locked worktrees,
   and detached-HEAD worktrees (skipped ones go in the report with the reason).
   For each candidate, verify **both**:
   - **Merged** — authoritative check via PR state:
     `gh pr list --head <branch> --state merged` returns the branch's PR. This
     is correct even for squash merges, which defeat ancestry checks. Only when
     `gh` or the remote is unavailable, fall back to
     `git merge-base --is-ancestor <branch> origin/<default-branch>`.
   - **Clean** — `git -C <worktree> status --porcelain` is empty AND the branch
     has no commits beyond the merged PR head (no unpushed work).

   If both hold, remove it:
   1. `git worktree remove <path>`
   2. `git branch -D <branch>` — `-D` is required because squash merges make
      `-d` refuse.
   3. `git push origin --delete <branch>` **only if** the remote branch still
      exists (GitHub's auto-delete usually beat us to it — check first, don't
      error on an already-gone remote branch).

   Anything failing a check is left untouched and recorded for the report with
   its reason: dirty / unmerged / unpushed commits / locked / detached.

3. **PM sweep.** Go through the pm-seam only
   ([`reference/pm-seam.md`](../../reference/pm-seam.md)): `findTasks`,
   `readTask`, `closeTask` — no new operations. If no PM is configured,
   skip this step silently (the same PM-optional posture as `/adlc:pr`).
   - Find issues that are **not** `Done`/`Canceled` and carry a task identity: a
     provider-linked branch (e.g. Linear's `gitBranchName`) or attached PR whose branch matches the task format
     `<initials>/<issue-identifier>-<slug>`.
   - For each candidate, resolve its PR's merge state — from the issue's PR
     attachment metadata when present, else
     `gh pr list --head <branch> --state merged`.
   - PR merged but status ≠ `Done` → `closeTask(taskRef)`. This deliberately
     jumps intermediate statuses — a documented exception to the seam's "never
     set a status out of order" rule, justified because cleanup is a safety net
     for tasks that already missed their lifecycle exits. Issues with an open or
     no PR are left untouched.

4. **Report.** Finish with one summary: worktrees removed (with the local/remote
   branches deleted alongside), issues moved to `Done`, and every skipped item
   with its reason. If nothing needed cleaning, say so — an immediate second run
   should remove nothing and report a clean state.

## Boundaries

- **Never** touch the main checkout, the current session's worktree, or any
  dirty, unmerged, unpushed, locked, or detached worktree — list them in the
  report instead.
- **Never** write card sections or labels — no `# Outcome`, no summaries;
  moving a missed issue to `Done` is a lightweight safety net behind the
  post-merge poller `/adlc:pr` spawns, nothing more.
- **Never** prompt per item — the sweep is fully automatic by design; the
  merged+clean criteria are the safety mechanism (each git/gh command still
  passes through the harness's normal permission prompts).
- **Never** delete a remote branch without first checking it still exists.

## Done when

`git worktree list` shows only the main checkout plus genuinely active work, no
merged-PR issues sit outside `Done`, and the user has the report of what was
removed, what was moved, and what was skipped and why.
