# adlc — Agent Guidelines

> A Claude Code plugin providing the ADLC (Agentic Development Life Cycle) harness — lifecycle skills, a Linear PM seam, and templates that scaffold agent-driven development into any repo.
>
> This file is the harness for AI coding agents working in this repo. It is loaded
> automatically. Rules here OVERRIDE default agent behavior — follow them exactly.
> Scaffolded by [ADLC](https://github.com/johngaynor/adlc); grow it as the project grows.

**Stack:** Markdown-only Claude Code plugin (skills, templates, reference docs) — no application code
**Package manager:** none (no dependencies)

---

## Task Router — read before you code

Match your task to one or more rows below and read those guides *first*. A task
often matches multiple rows; all apply. Only explore blindly when no row fits.

| Task | Guide |
|------|-------|
| Add or edit a lifecycle/utility skill | `skills/<name>/SKILL.md` — follow [`CONVENTIONS.md`](CONVENTIONS.md) exactly (trigger-style frontmatter description; purpose → arguments → workflow → boundaries → done-when) |
| Change anything about how skills talk to Linear | [`reference/pm-seam.md`](reference/pm-seam.md) owns all PM semantics (operations, statuses, labels, card layout) — change the seam first, then skills reference it |
| Change the card layout or artifact structure | [`reference/pm-seam.md`](reference/pm-seam.md) § Card Data Model / Artifact skeletons — skills never restate structure in prose |
| Change lifecycle vocabulary or methodology | [`METHODOLOGY.md`](METHODOLOGY.md) is the spine — Ask First; no skill redefines what the spine doesn't contain |
| Add or edit a file that init scaffolds into consuming repos | `templates/*.template` — keep placeholders in sync with `skills/init/SKILL.md` |
| Update user-facing docs | [`README.md`](README.md) — the skills table and repo-layout tree must match the actual filesystem |

> As areas grow their own rules, add a nested `CLAUDE.md` in that folder and route
> to it here rather than expanding this file indefinitely.

---

## Always

- Match the surrounding code: naming, structure, comment density, and idioms.
- Keep changes minimal, focused, and integrated through real call sites.
- Prefer the simplest change that fully solves the problem.
- Run the smallest relevant set of Validation Commands before claiming done.
- Capture recurring corrections as lessons — see the Self-Improvement section.
- Do PR-bound work in an isolated workspace (worktree): verify isolation first,
  provision one only if absent.
- Run `claude plugin validate .` after any edit to skills or plugin manifests.

## Ask First

- Before reducing scope, changing architecture, or changing a public contract.
- Before adding a production dependency.
- Before touching many files/modules in a way no spec or task covers.
- Before destructive or hard-to-reverse actions (deletes, migrations, force-push).
- Before editing the contract files — `METHODOLOGY.md`, `CONVENTIONS.md`,
  `reference/pm-seam.md` — they govern every skill in the plugin.

## Never

- Never introduce a temporary patch in place of a root-cause fix.
- Never commit secrets, credentials, tokens, or private keys.
- Never leave the build, typecheck, or tests broken.
- Never edit generated files by hand.
- Never commit directly to the main checkout — task work integrates through PRs
  from isolated workspaces.
- Never redefine pm-seam or METHODOLOGY vocabulary inside a skill — reference it.

## Validation Commands

Run the smallest set that proves your change:

```bash
claude plugin validate .
```

---

## Core Principles

- **Simplicity first** — every change as simple as possible; touch minimal code.
- **No laziness** — find root causes; senior-engineer standard; no band-aids.
- **Minimal impact** — change only what's necessary; avoid introducing risk.

## Workflow

- **Small fixes:** just fix them — no ceremony.
- **Non-trivial work (3+ steps or an architectural decision):** write a short spec
  first on the task's issue in the configured PM (`# Specification`), then
  implement. The `/adlc:brainstorm` → `/adlc:spec` skills drive this.
- **Verification:** run Validation Commands and confirm output before saying "done".
  Evidence before assertions.

## Self-Improvement (the lessons loop)

Recurring mistakes get written down, not repeated. When corrected on something that
will come up again, append a structured entry to [`.claude/lessons.md`](.claude/lessons.md)
(Context / Problem / Rule / Applies-to). Review that file at the start of a session.
The `/adlc:add-lesson` skill drives this.

Lessons climb a ladder as they mature: **correction → lesson → rule → skill**. A
recurring lesson graduates into a rule in this file; a lesson that describes a whole
repeatable workflow graduates into a project skill — the `/adlc:add-skill` skill
drives that.
<!-- TBD(project-skills): where project skills live and how they're discovered is
parked while the convention is redesigned; /adlc:add-skill asks where to write. -->
