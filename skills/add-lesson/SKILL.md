---
name: add-lesson
description: Use when the user corrects the agent on something likely to recur, or says "remember this", "add a lesson", "don't do that again". Writes a structured entry as a new file in .claude/lessons/ so the mistake isn't repeated.
---

# Add a lesson

Close the self-improvement loop: capture a correction as a durable, structured
lesson — one file per lesson in `.claude/lessons/`. See
[`METHODOLOGY.md`](../../METHODOLOGY.md) § Lessons loop.

## Arguments

- `$ARGUMENTS` (optional) — an explicit lesson to record. If absent, infer it from
  the correction in the conversation.

## Workflow

1. **Identify the lesson.** From `$ARGUMENTS` or the recent correction, extract the
   *generalizable* rule — the class of mistake, not the one-off instance.
2. **Sanity-check it's worth recording.** Skip if it's ephemeral conversational
   context or a one-time detail. Record only durable rules that will apply again. If
   it's unclear whether it generalizes, ask.
3. **Check for duplicates.** Read every file in `.claude/lessons/` (and check any
   legacy `.claude/lessons.md` if present). If an existing entry covers the same ground,
   **update** that file rather than adding a near-duplicate.
4. **Write the lesson file** at `.claude/lessons/<kebab-title>.md` — filename from
   the title, short and specific; on a collision append `-2`, `-3`, …:
   ```
   # <short, specific title>
   **Context**: where/when this came up.
   **Problem**: what went wrong.
   **Rule**: the durable rule that prevents it next time.
   **Applies to**: the paths or areas this governs.
   ```
   If `.claude/lessons/` doesn't exist, create it and add its `README.md` from
   `${CLAUDE_PLUGIN_ROOT}/templates/lessons-readme.md.template` (or reproduce the
   format doc inline if that path is not resolvable).
5. **Confirm** what was recorded and where.

## Boundaries

- **Never** record secrets, credentials, or one-off conversational context.
- **Never** append to a legacy `.claude/lessons.md` — new lessons always get their
  own file (updates to an existing legacy entry may edit it in place).
- **Ask First** only when it's genuinely unclear whether the correction generalizes.

## Done when

A well-formed, non-duplicate lesson file exists under `.claude/lessons/` and the
user has seen it.
