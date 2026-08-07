# PRD template

The standardized format for every PRD `/adlc:project` produces. The skill fills
this skeleton from the dialogue; it never invents sections beyond these.

## Name

The shippable chunk, named by its deliverable — short and concrete:

- ✅ `Weekly check-in flow`, `Coach dashboard`, `Offline logging`
- ❌ `Q3 improvements`, `Misc fixes`, `Training` (a domain that never
  completes is a label, not a project — see `reference/pm-seam.md`
  § Hierarchy)

## Description (the PRD)

```markdown
> <One-sentence outcome — what ships when this project is done.>

## Problem

<What hurts today and for whom — the gap this project closes.>

## Requirements

<Product-level requirements: what it must do, not how. Bullets.>

## User flows

<The key flows, step by step from the user's point of view.>

## Out of scope

<What this project deliberately does not cover.>

## Success criteria

- <Observable check #1 — something you could verify true or false>
- <Observable check #2>
- <…>
```

Rules:

- The blockquote opener is the glanceable one-liner — the same convention as an
  adlc card's preamble and an initiative's outcome statement, at the project
  level.
- **Product-level only.** No solution design, no phase breakdown, no issue
  list — engineering lives on the project's issues (their `# Specification` /
  `# Technical plan` blocks), and phasing becomes milestones at the breakdown
  stage. If a sentence tells an engineer *how* to build it, it doesn't belong
  in the PRD.
- Success criteria are observable checks, not vibes — they are what project
  updates report against, and what makes "is this shipped?" answerable.
