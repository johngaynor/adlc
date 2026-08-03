---
name: write-spec
description: Use when starting non-trivial work (roughly 3+ steps or an architectural decision) before writing code — the user asks to "write a spec", "plan this feature", or describes a sizable change. Produces a phased spec in .ai/specs/ from the ADLC spec template.
---

# Write a spec

Drive spec-first development: turn a feature brief into a short, phased spec at
`.ai/specs/{YYYY-MM-DD}-{kebab-title}.md` **before** any code is written. See
[`METHODOLOGY.md`](../../METHODOLOGY.md) § Spec-first for why.

## Arguments

- `$ARGUMENTS` (optional) — the feature brief. If absent, use the brief from the
  conversation, or ask the user for one.

## Workflow

1. **Gauge scope.** If the work is a small fix (roughly < 3 steps, no architectural
   decision), say so and recommend just doing it — do not manufacture a spec. Only
   continue for genuinely non-trivial work.
2. **Understand the brief.** Read `$ARGUMENTS` or the conversation. If the goal or
   success criteria are unclear, ask 1–3 focused questions before writing.
3. **Research the affected areas.** Route through the project's own Task Router (in
   the root `CLAUDE.md`) to find the relevant guides and code. Read enough to write
   a real Approach and honest Test Coverage — do not hand-wave.
4. **Derive filename parts.** Get today's date at runtime (`date +%F`) — never
   hardcode it. Make a short kebab-case title from the brief.
5. **Render the spec.** Copy `${CLAUDE_PLUGIN_ROOT}/templates/spec.md.template` (or
   reproduce its sections inline if unresolvable) into
   `.ai/specs/{date}-{title}.md`, creating `.ai/specs/` if needed. Fill: Problem/
   Goal, Approach (including rejected alternatives and blast radius), Phases, Test
   Coverage, Backward Compatibility.
6. **Surface Open Questions.** Present them to the user and keep `Status: draft`
   until they're resolved. Do not silently pick answers to material decisions.
7. **Report.** Give the created path and suggest the next move (implement, or
   `/adlc:ship-pr` once done).

## Boundaries

- **Ask First** before expanding scope beyond the brief.
- **Never** start implementing in this skill — it produces the plan only.

## Done when

A spec file exists with every section filled or explicitly marked `TODO`, and the
user has seen the Open Questions.
