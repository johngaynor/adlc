---
name: add-skill
description: Use when a repo-specific workflow deserves to become a project skill — the user says "make this a skill", "add a skill for this", or a lessons.md entry has grown into a whole multi-step workflow. Authors .ai/skills/<name>/SKILL.md in the target repo.
---

# Add a project skill

Author a skill that belongs to *this repo* — a committed, team-shared workflow in
`.ai/skills/<name>/SKILL.md`, the canonical project-skills home that Claude Code
discovers through the `.claude/skills` symlink (see
[`METHODOLOGY.md`](../../METHODOLOGY.md) § Extending the harness with project
skills). This is the top rung
of the self-improvement ladder (see [`METHODOLOGY.md`](../../METHODOLOGY.md)
§ Lessons loop): **correction → lesson → rule → skill**. `/adlc:add-lesson`
captures corrections; this skill graduates the ones that turn out to be whole
workflows.

## Arguments

- `$ARGUMENTS` (optional) — either a description of the workflow to capture, or
  the title of a `.claude/lessons.md` entry to graduate. If absent, infer the
  candidate from the conversation; ask if ambiguous.

## Workflow

1. **Identify the source.** From `$ARGUMENTS` or the conversation, determine
   which of the two entry paths applies:
   - **Fresh workflow** — the user is describing a repeatable procedure directly.
   - **Graduation** — an existing `.claude/lessons.md` entry has outgrown the
     lesson format and describes a multi-step workflow.
2. **Gate on skill-sized.** A repo skill is a *multi-step workflow* (how we run
   migrations, how we cut a release) with its own steps, boundaries, and
   definition of done. If the candidate is a one-line rule or a single fact, say
   so and stop — it belongs in `.claude/lessons.md` (via `/adlc:add-lesson`) or
   under a `CLAUDE.md` boundary heading, not here.
3. **Check for duplicates.** Read every existing `.ai/skills/*/SKILL.md` in
   the repo. If one already covers this ground, **update** it rather than adding
   a near-duplicate — and skip to step 7.
4. **Author the skill.** Render
   `${CLAUDE_PLUGIN_ROOT}/templates/skill.md.template` (reproduce its structure
   inline if the path is not resolvable) into
   `.ai/skills/<kebab-name>/SKILL.md`, filling every `{{PLACEHOLDER}}` and
   dropping the template's guidance comments:
   - `name` — kebab-case, matches the directory.
   - `description` — a trigger, not a title: "Use when…", written so an agent
     can decide to auto-invoke it.
   - Workflow — numbered, imperative steps the agent executes literally,
     including the repo's real commands and paths (never invented ones).
   - Boundaries — each rule under exactly one of Always / Ask First / Never.
   - Done when — how the agent proves the workflow succeeded.
5. **Graduation bookkeeping** (graduation path only). Rewrite the source lesson's
   body in `.claude/lessons.md` to a single pointer line —
   `**Graduated**: superseded by .ai/skills/<name>/ on <YYYY-MM-DD>` —
   keeping the entry's title. History stays greppable; guidance never lives in
   two places.
6. **Wire the router** (optional). If the new skill is the guide for a class of
   tasks, offer to add or update the matching Task Router row in the repo's root
   `CLAUDE.md` so tasks route to it.
7. **Verify and hand off.** Confirm the frontmatter parses (valid YAML, `name`
   matches the directory), no `{{…}}` placeholders remain, and the repo's
   `.claude/skills` symlink exists so Claude Code will discover the skill (if the
   repo predates the convention, point the user at re-running `/adlc:init`).
   Remind the user the skill loads in the *next* Claude Code session, and suggest
   committing it — project skills are shared with the team.

## Boundaries

- **Never** overwrite or restructure an existing repo skill without explicit
  confirmation — update it in place, and show what changed.
- **Never** record secrets, credentials, or machine-specific paths in a skill.
- **Never** invent commands or paths for the skill's workflow — every command a
  skill tells an agent to run must come from the repo's real configuration.
- **Ask First** when it is unclear whether the candidate generalizes to a
  repeatable workflow, or which lesson entry is meant.

## Done when

A valid `.ai/skills/<name>/SKILL.md` exists (frontmatter parses, no unfilled
placeholders), any graduated lesson points at it with its title intact, and the
user knows it loads next session and should be committed.
