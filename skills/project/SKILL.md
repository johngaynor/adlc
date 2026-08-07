---
name: project
description: Use when the user wants to create or define a project — one shippable chunk of work with its PRD ("create a project", "define the project for X", "write a PRD"). Runs a sharpening dialogue, then creates or retrofits the Linear project with the PRD as its description.
---

# Project

Define a **project** — one shippable chunk of work with a clear outcome and an
end — and produce its **PRD**, the product-level spec that lives as the
project's description. This is a planning-layer skill, sitting above the issue
lifecycle in the hierarchy defined in
[`reference/pm-seam.md`](../../reference/pm-seam.md) § Hierarchy: the
initiative holds the goal, the project holds the PRD, and the issues below it
hold the engineering. It talks to the PM through the seam's planning-tier
operations only, and creates no issues or milestones — decomposing the PRD is
the downstream breakdown stage.

## Arguments

- `projectRef` (optional) — an existing Linear project URL, ID, or slug. When
  given, this run is a **retrofit**: the project already exists and its
  description will be replaced with the approved PRD. When omitted, this run
  **creates** the project.

## Workflow

1. **Detect the mode and anchor to an initiative.**
   - **`projectRef` given (retrofit):** call `readProject(projectRef)` per
     `reference/pm-seam.md` and treat the existing description as input to the
     dialogue. Check its initiative membership: if unattached, call
     `listInitiatives()` and prompt the user to link one (they may decline);
     if attached, proceed.
   - **No `projectRef` (create):** open the dialogue by asking which
     initiative this project serves, offering the results of
     `listInitiatives()`. If one is named, call `readInitiative` and carry its
     outcome statement and success criteria into the dialogue as context, and
     attach the project to it at creation. If none, disregard and proceed
     unattached.
   - **No PM configured** (no `pm` block with `"provider": "linear"` in
     `.adlc/config.json` — the same detection the lifecycle stages use): the
     initiative question reads drafts under `.ai/product/initiatives/`
     instead; a missing PM never causes a refusal (see step 5).
2. **Apply the sizing gate.** Early on, test the idea against the decision
   test in `reference/pm-seam.md` § Hierarchy, in both directions: if it
   plausibly needs more than one shippable chunk, recommend
   `/adlc:initiative`; if it is issue-sized (one agent could spec → plan → PR
   it), recommend `/adlc:brainstorm`. The gate is soft — recommend and offer
   to stop, but the user may override and continue.
3. **Run the PRD dialogue.** One question at a time, per the same dialogue
   discipline as `/adlc:brainstorm`: clarify, never invent scope the user
   hasn't stated. Target the ingredients of the format in
   [`template.md`](template.md) — problem, requirements, user flows,
   out-of-scope, success criteria — until each clears the template's bar.
   Stop there: no solution design, no issue or milestone breakdown.
4. **Draft against the template.** Render the ingredients into the
   [`template.md`](template.md) format — Name plus templated Description — and
   present the draft for the user's approval. Iterate until approved. Never
   touch the PM or write a file before this approval.
5. **Deliver the approved PRD.**
   - **Linear configured:** call `saveProject` per `reference/pm-seam.md` —
     creating the project (with the initiative attach from step 1) or
     retrofitting the existing one's description. Hold the returned
     `projectRef` in session context only — no pointer file. If the user
     supplied supporting material too heavy for the description, attach it via
     `attachProjectDoc`; the PRD itself always stays in the description.
   - **No PM configured:** write the draft to
     `.ai/product/projects/<kebab-slug>.md` (slug derived from the Name),
     creating directories as needed, and report the path. This is the only
     file write this skill makes.
6. **Point downstream.** Report the project URL (or file path) and close by
   noting the next step: breaking the PRD down into issues and milestones —
   the breakdown stage, which consumes this PRD as its input.

## Boundaries

- **Never** write to the PM or the fallback file before the user approves the
  draft — and on a retrofit, the old description is dialogue input, so nothing
  is replaced sight unseen.
- **Never** put solution design, phase breakdowns, or issue lists in the PRD —
  engineering lives on the project's issues; phasing becomes milestones at the
  breakdown stage.
- **Never** create issues, milestones, or initiatives here — issues are the
  lifecycle's job, milestones the breakdown stage's, and initiative creation
  is app-only (`/adlc:initiative` drafts it).
- **Ask First** before widening scope beyond the chunk the user described.

## Done when

The approved PRD is the project's description — created or retrofitted via the
seam, attached to its initiative when one was chosen — and the user has the
project URL; or, with no PM configured, the PRD sits at
`.ai/product/projects/<kebab-slug>.md`. No issues or milestones were created
either way.
