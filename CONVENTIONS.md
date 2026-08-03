# Skill-Authoring Conventions (the contract)

> Every skill in this plugin follows these rules so that skills written in
> parallel read as one system. If you are filling in one of the stubbed skills,
> this file plus [`METHODOLOGY.md`](./METHODOLOGY.md) are your contract.

## File layout

- One skill per directory: `skills/<name>/SKILL.md`.
- `<name>` is kebab-case and becomes the invocation `/adlc:<name>`.
- Optional supporting files (`reference.md`, `scripts/`, `templates/`) live inside
  the skill's own directory unless they are shared across skills.

## Frontmatter

Every `SKILL.md` starts with YAML frontmatter:

```yaml
---
name: <kebab-case, matches the folder>
description: <one line, written so the agent can decide when to auto-invoke it — start with "Use when...">
---
```

`description` is the *only* thing the agent sees when deciding whether a skill
applies. Make it a trigger, not a title: "Use when the user asks to open a PR..."
beats "PR helper".

## Body structure

Keep bodies imperative and skimmable. Preferred sections, in order:

1. **One-paragraph purpose** — what this skill does and when.
2. **Arguments** — if the skill takes `$ARGUMENTS`, list them.
3. **Workflow** — numbered steps the agent executes literally. This is the meat.
4. **Boundaries** — any Always / Ask First / Never specific to this skill.
5. **Done when** — the definition of done and how to validate.

## Voice & rules

- Second person, imperative: "Read the spec.", "Run the tests.", not "You could...".
- Reference the shared vocabulary (Always / Ask First / Never / Validation, Task
  Router, spec-first, lessons loop) — do **not** redefine it inline. Link to
  `METHODOLOGY.md`.
- Skills may read and write files in the user's working directory via the normal
  tools; they are subject to the user's permission prompts. Never bypass them.
- Idempotency: a skill that scaffolds files must be safe to run twice — detect
  existing files and ask before overwriting.
- No hardcoded absolute paths, no assumptions about the user's stack beyond what
  the skill detects or asks for.

## Namespacing

All skills auto-namespace as `/adlc:<name>`. Don't prefix names yourself.
