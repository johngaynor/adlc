---
name: breakdown-projects
description: Use when an initiative needs its contributing projects enumerated — "break the initiative into projects", "enumerate the chunks", "what projects serve this goal". Runs a dialogue proposing the shippable chunks, then creates them as initiative-attached shell projects whose PRDs come later via /adlc:project.
---

# Breakdown projects

Decompose an initiative's goal into its contributing **projects** — the chunk
enumeration `/adlc:initiative` deliberately excludes. Each chunk becomes a
**shell project**: a name and a one-liner, attached to the initiative, carrying
a pending-marker instead of a PRD. Shells are deliberately thin — the full PRD
is `/adlc:project`'s job, run per shell when its turn comes. This is a
planning-layer skill operating under the hierarchy in
[`reference/pm-seam.md`](../../reference/pm-seam.md) § Hierarchy; it talks to
the PM through the seam's planning-tier operations only.

## Arguments

- `initiativeRef` (optional) — the initiative's name or slug, threaded in when
  invoked from `/adlc:initiative`'s closing hand-off. When standalone and
  omitted, resolve it via `listInitiatives()` per `reference/pm-seam.md` and
  ask the user which initiative to break down (or for the exact name, when the
  initiative is too new to be listed).

## Workflow

1. **Resolve the initiative.** Per the Arguments rule. Load its outcome
   statement and success criteria as dialogue context via `readInitiative` —
   with no PM configured, the initiative may instead be a draft under
   `.ai/product/initiatives/`; read that.
2. **Enumerate the chunks in dialogue.** One question at a time, propose and
   refine the contributing projects. Each chunk gets exactly a **name** and a
   **one-liner** (its outcome, one sentence). Hold every chunk to the project
   bar of the seam's Hierarchy decision test: a deliverable with an end — a
   domain that never completes belongs to labels, and something issue-sized
   belongs in `/adlc:brainstorm`. Do not write PRD content here: no
   requirements, no user flows — that depth is `/adlc:project`'s.
3. **Present the breakdown for approval.** The full list — chunk names,
   one-liners, and the initiative they attach to — approved as a set before
   any write. Iterate until approved.
4. **Create the shells.**
   - **Linear configured:** one `saveProject` per chunk, per
     `reference/pm-seam.md` — description = the one-liner plus the marker
     `PRD pending — run /adlc:project <name>`, attached to the initiative.
     Verify and report each attach result; a failed attach (name mismatch) is
     reported with the corrective step (paste the exact initiative name),
     never silently dropped.
   - **No PM configured** (no `pm` block with `"provider": "linear"` in
     `.adlc/config.json`): write each shell as a stub at
     `.ai/product/projects/<kebab-slug>.md` — the one-liner plus the same
     pending-marker — creating directories as needed.
5. **Report and point downstream.** List the created shells (URLs or file
   paths) and close by noting the next step: run `/adlc:project <shell>` to
   retrofit each shell with its full PRD, starting with whichever chunk the
   user wants to build first.

## Boundaries

- **Never** write to the PM or the fallback files before the user approves the
  full breakdown.
- **Never** write PRD content — shells carry a name, a one-liner, and the
  pending-marker, nothing more; depth belongs to `/adlc:project`.
- **Never** create issues, milestones, or initiatives here — this skill only
  creates shell projects and attaches them.
- **Ask First** before widening the breakdown beyond the initiative's stated
  goal.

## Done when

Every approved chunk exists as a shell project attached to the initiative (or
as a stub under `.ai/product/projects/` with no PM), each attach result is
reported, and the user knows the next step is `/adlc:project` per shell.
