---
name: add-lesson
description: Use when the user corrects the agent on something likely to recur, or says "remember this", "add a lesson", "don't do that again". Appends a structured entry to .claude/lessons.md so the mistake isn't repeated.
---

# Add a lesson

> ⚠️ **STUB — Stream D.** This skill is scaffolded but not yet implemented. Fill in
> the Workflow below following [`CONVENTIONS.md`](../../CONVENTIONS.md) and
> [`METHODOLOGY.md`](../../METHODOLOGY.md) (§ Lessons loop). Remove this banner when done.

Close the self-improvement loop: capture a correction as a durable, structured
lesson in `.claude/lessons.md`.

## Build contract (what this skill must do)

1. Infer the lesson from the correction in the conversation (or `$ARGUMENTS`).
2. Write it in the standard format — **Context / Problem / Rule / Applies-to** — with
   a short, specific title.
3. Append to `.claude/lessons.md` (create it from `templates/lessons.md.template` if
   missing). Do not duplicate an existing lesson — if one covers the same ground,
   update it instead.
4. Keep the rule general enough to prevent the class of mistake, specific enough to
   act on. Cite the paths it applies to.
5. Confirm what was recorded.

**Boundaries:** Never record secrets or one-off conversational context as a lesson —
only durable rules. Ask First only if it's unclear whether the correction generalizes.

**Done when:** a well-formed entry exists in `.claude/lessons.md` and the user has
seen it.
