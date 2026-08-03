---
name: add-lesson
description: Use when the user corrects the agent on something likely to recur, or says "remember this", "add a lesson", "don't do that again". Appends a structured entry to .claude/lessons.md so the mistake isn't repeated.
---

# Add a lesson

Close the self-improvement loop: capture a correction as a durable, structured
lesson in `.claude/lessons.md`. See [`METHODOLOGY.md`](../../METHODOLOGY.md)
§ Lessons loop.

## Arguments

- `$ARGUMENTS` (optional) — an explicit lesson to record. If absent, infer it from
  the correction in the conversation.

## Workflow

1. **Identify the lesson.** From `$ARGUMENTS` or the recent correction, extract the
   *generalizable* rule — the class of mistake, not the one-off instance.
2. **Sanity-check it's worth recording.** Skip if it's ephemeral conversational
   context or a one-time detail. Record only durable rules that will apply again. If
   it's unclear whether it generalizes, ask.
3. **Check for duplicates.** Read `.claude/lessons.md`. If an existing entry covers
   the same ground, **update** it rather than adding a near-duplicate.
4. **Write the entry** in the standard format, with a short, specific title:
   ```
   ## <short, specific title>
   **Context**: where/when this came up.
   **Problem**: what went wrong.
   **Rule**: the durable rule that prevents it next time.
   **Applies to**: the paths or areas this governs.
   ```
5. **Append** to `.claude/lessons.md` (create it from
   `${CLAUDE_PLUGIN_ROOT}/templates/lessons.md.template`, or with the header inline,
   if it doesn't exist).
6. **Confirm** what was recorded and where.

## Boundaries

- **Never** record secrets, credentials, or one-off conversational context.
- **Ask First** only when it's genuinely unclear whether the correction generalizes.

## Done when

A well-formed, non-duplicate entry exists in `.claude/lessons.md` and the user has
seen it.
