# Worktree isolation as a core ADLC idea + repo permission config

**Date:** 2026-08-03
**Status:** Approved design, pending implementation

## What

Two related additions, shipped as two PRs:

1. **Workspace isolation becomes ADLC idea #5** — a detection-first invariant
   ("PR-bound work happens in an isolated workspace") woven through
   METHODOLOGY.md, the ship-pr skill, the CLAUDE.md template, the init skill,
   and the README.
2. **This repo gets a committed `.claude/settings.json`** allowlisting git/gh
   operations so routine GitHub work never prompts, with guards on destructive
   commands.

## Why

The goal of ADLC is hands-off operation with multiple agents running in
parallel. Parallelism is only safe when agents never share mutable state —
each task needs its own isolated workspace, integrated through PRs. Today the
methodology is silent on this, and `ship-pr` actively misbehaves in a worktree
(it ships PRs from machine-generated branch names like
`worktree-ancillary-config`).

Platforms differ: Conductor drops the agent into a worktree before it wakes
up; Claude Code has a native worktree tool; bare harnesses have only git.
So the methodology mandates the **invariant, not the mechanism**, and uses
**detection over configuration** — no platform registry to maintain, and new
platforms with built-in isolation work automatically.

## PR 1: Worktree methodology (new worktree, ~5 files)

### METHODOLOGY.md — idea #5: "Isolated parallel work"

Core content:

- **Invariant:** one task = one worktree (isolated workspace) = one PR. The
  main checkout is never edited by agents; the human gate is PR merge.
- **Detection-first rule:** before starting PR-bound work, verify isolation;
  provision only if absent. Detection:
  `git rev-parse --git-dir` ≠ `git rev-parse --git-common-dir` → already in a
  linked worktree → do nothing. (Guard: `git rev-parse
  --show-superproject-working-tree` returning a path means submodule, not
  worktree.)
- **Provisioning order:** platform-provided isolation (already detected) →
  harness-native worktree tool → fallback `git worktree add
  .worktrees/<task-slug>` with `.worktrees/` gitignored.
- **Lifecycle:** implement and validate inside the worktree; ship a PR from
  it; after the PR is pushed the worktree is disposable (branch lives on the
  remote). Human reviews and merges.
- Applies to **every PR-bound task**, small fixes included.

### skills/ship-pr/SKILL.md — worktree-aware branching

- Replace "if on the default branch, create a feature branch" with:
  - On a machine-generated branch (matches `worktree-*` or similar): rename /
    recreate as a proper kebab-case feature branch before pushing.
  - On a sensibly-named branch (platform-managed, e.g. Conductor): leave it
    alone.
  - On the default branch (defensive case): create a feature branch as today.
- After opening the PR, note that the worktree can be removed.

### templates/CLAUDE.md.template — boundary rules

- **Always:** "PR-bound work happens in an isolated workspace (worktree);
  verify isolation before starting, provision only if absent."
- **Never:** "Never commit directly to the main checkout while task work is
  in flight."

### skills/init — repo hygiene

- Ensure `.worktrees/` is present in the target repo's `.gitignore`.

### README.md

- Methodology summary gains idea #5 (one line, consistent with the others).

## PR 2: Repo permission config (this worktree: `ancillary-config`)

Committed `.claude/settings.json` in this repo:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(gh *)"
    ],
    "ask": [
      "Bash(gh repo delete *)",
      "Bash(gh secret *)",
      "Bash(gh api --method DELETE *)",
      "Bash(gh api -X DELETE *)"
    ]
  }
}
```

Rationale: physiq's settings.local.json accumulated ~330 one-off allow rules
through prompt fatigue and converged on `git *` / `gh *` anyway. Committed
project settings make the intent durable and reviewable; `ask` guards keep a
speed bump on destructive operations (ask rules take precedence over broader
allow rules).

## Out of scope

- Auto-merge on green CI (human merges; revisit later).
- Per-platform adaptation docs (detection makes them unnecessary for now).
- Broader non-GitHub permission allowlists, CI workflows, PR templates
  (discussed, deferred).

## Testing / validation

- PR 1 is documentation/skill-text only: validation is reading the rendered
  files for consistency (vocabulary defined in METHODOLOGY.md, quoted
  elsewhere — per the "spine" rule).
- Detection commands verified manually in three states: main checkout, linked
  worktree, submodule.
- PR 2: `claude` session in this repo confirms gh/git commands no longer
  prompt and `gh repo delete` still does.
