---
name: write-spec
description: Use when starting non-trivial work (roughly 3+ steps or an architectural decision) before writing code — the user asks to "write a spec", "plan this feature", or describes a sizable change. Produces a phased spec in .ai/specs/ from the ADLC spec template.
---

# Write a spec

> ⚠️ **STUB — Stream B.** This skill is scaffolded but not yet implemented. Fill in
> the Workflow below following [`CONVENTIONS.md`](../../CONVENTIONS.md) and
> [`METHODOLOGY.md`](../../METHODOLOGY.md) (§ Spec-first). Remove this banner when done.

Drive spec-first development: turn a feature brief into a short, phased spec at
`.ai/specs/{YYYY-MM-DD}-{kebab-title}.md` before any code is written.

## Build contract (what this skill must do)

1. Take the user's brief (from `$ARGUMENTS` or the conversation).
2. Research the affected areas of the codebase enough to write a real approach —
   route the research through the project's own Task Router.
3. Derive a kebab-case title and today's date for the filename. (The date must be
   obtained at runtime — do not hardcode.)
4. Render `templates/spec.md.template` (in this plugin) into `.ai/specs/`, filling
   Problem/Goal, Approach, Phases, Test Coverage, Backward Compatibility.
5. Ask the user the Open Questions before finalizing status beyond `draft`.
6. Report the created path and suggest `/adlc:ship-pr` or implementation next.

**Boundaries:** Ask First before expanding scope beyond the brief. Never start
implementing in this skill — it produces the plan only.

**Done when:** a spec file exists with all sections filled or explicitly marked
`TODO`, and the user has seen the Open Questions.
