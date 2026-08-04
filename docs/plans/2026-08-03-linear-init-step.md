# Linear Setup Step in `/adlc:init` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional "Connect Linear" step to the `/adlc:init` skill that verifies the Linear MCP with a live call, walks the user through team and project selection, and writes the mapping to a committed `.adlc/config.json`.

**Architecture:** One prose change to `skills/init/SKILL.md` — a new workflow step 5 between "Write the files" and the closing report, plus matching Boundaries and "Done when" additions. The config surface (`.adlc/config.json` with a `pm` block) is shared with the lifecycle plan's PM seam; this supersedes Task 8 of `docs/plans/2026-08-03-linear-lifecycle.md`. Spec: `.ai/specs/2026-08-03-linear-init-integration.md`.

**Tech Stack:** Claude Code plugin skill (`SKILL.md` prose, YAML frontmatter). No executable code; validation is by grep.

## Global Constraints

- Follow `CONVENTIONS.md`: imperative second person; body order purpose → workflow → boundaries → done-when; reference `METHODOLOGY.md` vocabulary without redefining it.
- Idempotency: scaffolding must be safe to run twice — detect existing files and ask before overwriting.
- No hardcoded absolute paths.
- Config shape (exact): `{ "pm": { "provider": "linear", "team": "<team key>", "teamName": "<team name>", "project": "<project id>", "projectName": "<project name>" } }`.
- Gitignore entries (exact): `.adlc/*` and `!.adlc/config.json`.

---

### Task 1: Add the "Connect Linear" step to `skills/init/SKILL.md`

**Files:**
- Modify: `skills/init/SKILL.md`

**Interfaces:**
- Consumes: existing init workflow steps 1–5 (step 5 is "Report and hand off", to be renumbered 6).
- Produces: init writes `.adlc/config.json` with the `pm` block above; `.gitignore` entries; a documented offer/verify/team/project flow later skills can rely on.

- [ ] **Step 1: Insert the new workflow step**

In `skills/init/SKILL.md`, retitle `### 5. Report and hand off` to `### 6. Report and hand off`, and insert before it:

```markdown
### 5. Connect Linear (optional)

Offer to connect this repo to Linear for issue tracking. If the user declines,
skip this step entirely — write nothing Linear-related.

If they accept:

1. **Verify the Linear MCP with a live call** — list the workspace's teams via
   the Linear MCP. If the MCP is not configured or the call fails, give the
   user exact setup instructions for adding the Linear MCP server, then
   continue init without Linear — they can re-run `/adlc:init` later to
   retrofit just this step.
2. **Choose the team** — exactly one team returned: use it and say which.
   Multiple teams: ask the user to pick one.
3. **Choose the project** — always ask. List the team's existing projects and
   ask the user to select one **or** create a new one (default the new
   project's name to the repo name). Create it via the MCP if requested.
4. **Write `.adlc/config.json`** in the repo root:

   ```json
   {
     "pm": {
       "provider": "linear",
       "team": "<team key>",
       "teamName": "<team name>",
       "project": "<project id>",
       "projectName": "<project name>"
     }
   }
   ```

   If `.adlc/config.json` already exists with a `pm` block, show it and ask
   before changing it.
5. **Gitignore** — ensure the repo's `.gitignore` contains `.adlc/*` and
   `!.adlc/config.json` (the config is shared; anything else under `.adlc/`
   is per-machine). Add the lines only if missing.

If any MCP call fails mid-flow, report it plainly, write nothing partial, and
finish the rest of init normally. Never write a config containing unverified
IDs.
```

- [ ] **Step 2: Extend Boundaries and Done when**

In the same file, add to the `## Boundaries` list:

```markdown
- **Never** write `.adlc/config.json` with team or project IDs that did not
  come from a live Linear MCP response.
- **Ask First** before replacing an existing `pm` block in `.adlc/config.json`.
```

And extend the `## Done when` paragraph so it also covers: if the user opted
into Linear, `.adlc/config.json` exists with the `pm` block, the IDs came from
live MCP responses, and `.gitignore` covers `.adlc/`.

- [ ] **Step 3: Validate**

Run: `grep -c '\.adlc/config\.json\|Linear MCP\|Connect Linear' skills/init/SKILL.md`
Expected: ≥ 5 (step title, config path in workflow + boundaries + done-when, MCP verification present).

Run: `grep -n '### [0-9]' skills/init/SKILL.md`
Expected: steps numbered 1–6 in order with no duplicates.

- [ ] **Step 4: Commit**

```bash
git add skills/init/SKILL.md
git commit -m "feat: add optional Linear setup step to /adlc:init"
```
