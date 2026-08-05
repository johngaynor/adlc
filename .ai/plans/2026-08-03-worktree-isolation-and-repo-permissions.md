# Worktree Isolation + Repo Permissions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make workspace isolation (detection-first worktrees) ADLC methodology idea #5 across all plugin docs/skills, and add a committed permissions allowlist to this repo.

**Architecture:** Two independent PRs from two worktrees. PR 2 (repo permissions) is built in the existing `ancillary-config` worktree and shipped first since the session is already there. PR 1 (methodology) is built in a second worktree created afterwards. METHODOLOGY.md is "the spine": the new vocabulary is defined there first, then quoted (never redefined) in ship-pr, the CLAUDE.md template, init, and README.

**Tech Stack:** Markdown + JSON only. No test suite exists; validation is `jq` parses + `grep` consistency checks.

**Spec:** `.ai/specs/2026-08-03-worktree-isolation-and-repo-permissions.md`

## Global Constraints

- Vocabulary is defined in METHODOLOGY.md first, then propagated — skills/templates quote it, never redefine it (the "spine" rule).
- Commit messages: conventional commits, ending with the trailer:
  `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` and the Claude-Session line used by earlier commits in this session.
- The invariant phrase is exactly: **one task = one isolated workspace = one PR** (use "isolated workspace", not "worktree", when naming the invariant; worktrees are the mechanism).
- Branch names: PR 2 → `feat/repo-permissions`; PR 1 → `feat/worktree-methodology`.
- Never run bare `git stash`; worktrees share the stash stack.

---

### Task 1: Committed permissions for this repo (PR 2 content)

**Files:**
- Create: `.claude/settings.json` (in the `ancillary-config` worktree = repo root of that checkout)
- Verify: `.gitignore`

**Interfaces:**
- Produces: the permissions file PR 2 ships. Nothing downstream consumes it.

- [ ] **Step 1: Write the settings file**

Create `.claude/settings.json` with exactly:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
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

- [ ] **Step 2: Validate JSON parses and rules are present**

Run: `jq -e '.permissions.allow | length == 2' .claude/settings.json && jq -e '.permissions.ask | length == 4' .claude/settings.json`
Expected: `true` twice, exit 0.

- [ ] **Step 3: Ensure local-only settings stay ignored**

Run: `cat .gitignore`
If it does not already contain `.claude/settings.local.json`, append that line (settings.json is committed; settings.local.json must never be). Also confirm `.claude/worktrees/` is not trackable — if missing, append `.claude/worktrees/`.

- [ ] **Step 4: Commit**

```bash
git add .claude/settings.json .gitignore
git commit -m "chore: commit project permission allowlist for git/gh"
```

(with the required trailer lines)

---

### Task 2: Ship PR 2

**Files:** none (git/gh operations only)

**Interfaces:**
- Consumes: Task 1's commit plus the already-committed spec and plan on this branch.

- [ ] **Step 1: Rename the machine-generated branch** (dogfooding the new methodology)

```bash
git branch -m worktree-ancillary-config feat/repo-permissions
```

- [ ] **Step 2: Push and open the PR**

```bash
git push -u origin feat/repo-permissions
gh pr create --title "chore: project permission allowlist + ancillary-config spec/plan" --body "<what/why/validation per ship-pr contract>"
```

Body must state: what changed (committed `.claude/settings.json` allowlist with ask-guards; spec + plan docs), why (kill prompt fatigue for git/gh while keeping a speed bump on destructive ops), how validated (jq checks), and follow-ups (PR 1 in flight).

- [ ] **Step 3: Report the PR URL to the user.**

---

### Task 3: Provision the methodology worktree and switch into it

**Files:** none (worktree operations)

**Interfaces:**
- Produces: a second worktree at `.claude/worktrees/worktree-methodology` on branch `feat/worktree-methodology`, session switched into it. All Task 4–9 paths are relative to it.

- [ ] **Step 1: Create the worktree from origin/main**

```bash
git -C /Users/johngaynor/Dev/adlc worktree add /Users/johngaynor/Dev/adlc/.claude/worktrees/worktree-methodology -b feat/worktree-methodology origin/main
```

- [ ] **Step 2: Switch the session into it**

Use the native tool: `EnterWorktree` with `path: /Users/johngaynor/Dev/adlc/.claude/worktrees/worktree-methodology`.
(Note: after this switch the `ancillary-config` worktree is no longer writable from this session — PR 2 must already be shipped.)

---

### Task 4: METHODOLOGY.md — idea #5

**Files:**
- Modify: `METHODOLOGY.md` (line 13 and insert new section between idea 4 and "The three core principles")

**Interfaces:**
- Produces: the canonical vocabulary — "Isolated parallel work", the invariant "one task = one isolated workspace = one PR", the 3-step detect/provision/fallback procedure, and the ship-time branch-rename rule. Tasks 5–8 quote these; they must not restate the detection commands.

- [ ] **Step 1: Update the idea count**

Change line 13 from `The methodology is four ideas. Adopt them in order; each is useful on its own.` to `The methodology is five ideas. Adopt them in order; each is useful on its own.`

- [ ] **Step 2: Insert the new section**

Insert after the lessons-loop section (after line 88's paragraph ending "...reviewed at session start." and its following `---`), before `## The three core principles`:

```markdown
## 5. Isolated parallel work

The unit of parallelism: **one task = one isolated workspace = one PR.** An agent
never edits the main checkout — it stays on the default branch as the stable
reference copy. Every PR-bound task, however small, happens in its own isolated
workspace (a git worktree or equivalent) and integrates through a PR a human
reviews and merges. With no shared mutable state before merge, any number of
agents can work in parallel.

The rule is the **invariant, not the mechanism** — detection over configuration:

1. **Detect.** Before starting PR-bound work, check whether you are already
   isolated: if `git rev-parse --git-dir` and `git rev-parse --git-common-dir`
   differ, you are in a linked worktree — proceed, create nothing. (Guard: if
   `git rev-parse --show-superproject-working-tree` prints a path, you are in a
   submodule, not a worktree.) Platforms with built-in worktrees (e.g.
   Conductor) pass this check automatically — zero configuration.
2. **Provision natively.** Not isolated? Use the harness's native worktree
   tool if one exists (e.g. Claude Code's built-in worktree support).
3. **Fall back to git.** Otherwise: `git worktree add .worktrees/<task-slug>`,
   with `.worktrees/` in `.gitignore`.

**Lifecycle:** implement and validate inside the workspace; ship a PR from it;
once the PR is pushed the workspace is disposable — the branch lives on the
remote. At ship time, machine-generated branch names (e.g. `worktree-*`) are
renamed to proper feature branches; sensible platform-chosen names are kept.

---
```

- [ ] **Step 3: Verify consistency**

Run: `grep -c "five ideas" METHODOLOGY.md && grep -c "one task = one isolated workspace = one PR" METHODOLOGY.md`
Expected: `1` and `1`.

- [ ] **Step 4: Commit**

```bash
git add METHODOLOGY.md
git commit -m "feat: add idea #5 — isolated parallel work (detection-first worktrees)"
```

---

### Task 5: ship-pr build contract — worktree-aware branching

**Files:**
- Modify: `skills/ship-pr/SKILL.md` (contract item 2, and the "Done when" line)

**Interfaces:**
- Consumes: Task 4's vocabulary (quote, don't redefine).

- [ ] **Step 1: Replace contract item 2**

Replace:

```markdown
2. If on the default branch, create a feature branch first (kebab-case, prefixed by
   change type, e.g. `feat/`, `fix/`, `chore/`).
```

with:

```markdown
2. Get on a proper feature branch (kebab-case, prefixed by change type, e.g.
   `feat/`, `fix/`, `chore/`) per METHODOLOGY.md idea 5:
   - On a machine-generated branch (e.g. `worktree-*`): rename it
     (`git branch -m <new-name>`) before pushing.
   - On the default branch: create the feature branch first.
   - On a sensibly-named branch already (e.g. platform-managed): keep it.
```

- [ ] **Step 2: Extend item 5 and "Done when"**

Change item 5 from `5. Report the PR URL.` to:

```markdown
5. Report the PR URL. If working in an isolated workspace (worktree), note that
   it is now disposable — the branch lives on the remote.
```

- [ ] **Step 3: Verify**

Run: `grep -c "worktree-\*" skills/ship-pr/SKILL.md`
Expected: `1` (or more).

- [ ] **Step 4: Commit**

```bash
git add skills/ship-pr/SKILL.md
git commit -m "feat: make ship-pr contract worktree-aware (branch rename rule)"
```

---

### Task 6: CLAUDE.md.template — boundary rules

**Files:**
- Modify: `templates/CLAUDE.md.template` (Always and Never sections)

**Interfaces:**
- Consumes: Task 4's vocabulary.

- [ ] **Step 1: Add the Always rule**

Insert before the `{{ALWAYS_EXTRA}}` line:

```markdown
- Do PR-bound work in an isolated workspace (worktree): verify isolation first,
  provision one only if absent.
```

- [ ] **Step 2: Add the Never rule**

Insert before the `{{NEVER_EXTRA}}` line:

```markdown
- Never commit directly to the main checkout — task work integrates through PRs
  from isolated workspaces.
```

- [ ] **Step 3: Verify placeholders intact**

Run: `grep -c "{{ALWAYS_EXTRA}}" templates/CLAUDE.md.template && grep -c "{{NEVER_EXTRA}}" templates/CLAUDE.md.template`
Expected: `1` and `1`.

- [ ] **Step 4: Commit**

```bash
git add templates/CLAUDE.md.template
git commit -m "feat: add isolation boundary rules to CLAUDE.md template"
```

---

### Task 7: init skill — gitignore hygiene

**Files:**
- Modify: `skills/init/SKILL.md` (step 4 "Write the files" list, and "Done when")

**Interfaces:**
- Consumes: Task 4's fallback path convention (`.worktrees/`).

- [ ] **Step 1: Add the gitignore bullet**

In section `### 4. Write the files`, after the `.ai/specs/README.md` bullet, add:

```markdown
- **`.gitignore`** — ensure a `.worktrees/` entry exists (append to the existing
  file, or create it), so fallback worktrees (METHODOLOGY.md idea 5) are never
  committed.
```

- [ ] **Step 2: Update "Done when"**

Change the "Done when" sentence to also require the `.gitignore` entry:
`CLAUDE.md`, `.claude/lessons.md`, `.ai/specs/`, and a `.worktrees/` gitignore entry exist, ...

- [ ] **Step 3: Commit**

```bash
git add skills/init/SKILL.md
git commit -m "feat: init ensures .worktrees/ is gitignored"
```

---

### Task 8: README — five ideas

**Files:**
- Modify: `README.md` (line 10)

**Interfaces:**
- Consumes: Task 4's idea count.

- [ ] **Step 1: Update the count**

Change `for the four ideas it's built on.` to `for the five ideas it's built on.`

- [ ] **Step 2: Verify no stale counts remain anywhere**

Run: `grep -rn "four ideas" README.md METHODOLOGY.md CONVENTIONS.md templates/ skills/`
Expected: no output (exit 1).

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: README reflects five methodology ideas"
```

---

### Task 9: Ship PR 1

**Files:** none

**Interfaces:**
- Consumes: Tasks 4–8 commits on `feat/worktree-methodology`.

- [ ] **Step 1: Final consistency check**

Run: `grep -rn "one task = one isolated workspace = one PR" . --include="*.md" | grep -v .ai/`
Expected: exactly one hit — METHODOLOGY.md (the spine defines it; others reference idea 5 without restating).

- [ ] **Step 2: Push and open the PR**

```bash
git push -u origin feat/worktree-methodology
gh pr create --title "feat: isolated parallel work — methodology idea #5" --body "<what/why/validation + link to spec in PR 2>"
```

Body: what (idea #5 + ship-pr/template/init/README propagation), why (parallel agents need isolation; detection-first keeps it platform-neutral), validation (grep consistency checks), reference the spec file from PR 2's branch.

- [ ] **Step 3: Report both PR URLs to the user.**
