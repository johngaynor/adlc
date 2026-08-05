# Linear setup in `/adlc:init`

**Date**: 2026-08-03
**Status**: approved design, not yet implemented
**Supersedes**: Task 8 of `docs/plans/2026-08-03-linear-lifecycle.md` (this spec
is the richer version of that task's "PM integration" step; the config surface
is shared)

## What

Add an optional Linear setup step to the `/adlc:init` skill. It connects the
repo being initialized to a Linear project (one shared Linear workspace/team
across many repos; each repo gets its own project — e.g. the `physiq` repo maps
to a `physiq` project in Linear), and records that mapping in a committed
`.adlc/config.json` — the same config surface the ADLC lifecycle's PM seam
reads.

## Why

Linear is the source of work for these repos: issues are planned in Linear
(by hand or, later, with Claude's help) and picked up from there. For any skill
to operate on "this repo's issues," the repo needs a durable, committed record
of which Linear team and project it belongs to. `.adlc/config.json` is that
record: machine-readable for the PM seam, committed so every clone and
collaborator resolves it without setup.

## Design

### Placement

A new step in `skills/init/SKILL.md`, inserted between the current step 4
("Write the files") and step 5 ("Report and hand off"): **step 5 — Connect
Linear (optional)**. The existing report step becomes step 6. No new skill and
no new config file.

### Flow

1. **Offer** — ask the user whether to connect this repo to Linear for issue
   tracking. If declined, skip entirely; nothing Linear-related is written.
2. **Verify the MCP is live** — make a real Linear MCP call (list teams). If
   the MCP is not configured or the call fails, tell the user how to add the
   Linear MCP server and continue init without Linear; they can re-run init
   later to retrofit this step.
3. **Team** — from the teams returned: exactly one team → use it and state
   which; multiple teams → ask the user to pick one.
4. **Project** — always ask. List the existing projects in the chosen team and
   ask the user to either select one or create a new one (default the
   new-project name to the repo name). Create via the MCP if requested.
5. **Write the mapping** — write `.adlc/config.json` in the target repo:

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

   The `provider` and `team` fields match what the lifecycle plan's Task 8
   defined; `project`/`projectName` extend it with the repo → project mapping.
6. **Gitignore** — ensure `.adlc/` is ignored *except* the shared config:
   append `.adlc/*` and `!.adlc/config.json` to the target repo's
   `.gitignore` (config is shared; any future task cache under `.adlc/` is
   not).

### Idempotency

Follows init's existing conflict rules: if `.adlc/config.json` already exists
with a `pm` block, show it and ask before changing it. Re-running init on a
repo that is already scaffolded but has no PM config offers just this step as
the retrofit path.

### Error handling

If the MCP is unreachable or a call fails mid-flow: report it plainly, write
nothing partial, and finish the rest of init normally. Never write a config
containing unverified IDs.

## Files touched

- `skills/init/SKILL.md` — add the new step; renumber; extend Boundaries and
  "Done when" to cover the optional PM config.
- `templates/CLAUDE.md.template` — no change; the mapping lives in
  `.adlc/config.json`, not `CLAUDE.md`.

## Out of scope (deferred)

- The issue lifecycle: pulling/starting issues, branch naming, status updates.
- The spec ↔ issue relationship (how `.ai/specs/` files relate to Linear
  issues).
- Claude-driven board population (decomposing a goal into Linear issues).
- Any always-on MCP reachability rule in `CLAUDE.md` (explicitly dropped; the
  MCP is verified once, during init).
- The rest of the lifecycle plan (`docs/plans/2026-08-03-linear-lifecycle.md`)
  — only its Task 8 is superseded by this spec.
