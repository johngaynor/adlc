# ADLC — AI Development Life Cycle

A portable engineering harness for AI coding agents, packaged as a **Claude Code
plugin**. Install it into any repo to get an opinionated `CLAUDE.md` (with a Task
Router and boundary-labeled rules), plus skills for spec-first development, PR
discipline, and a self-improvement lessons loop.

The methodology is deliberately framework-agnostic — it works on a TypeScript
monorepo, a Python service, or a mobile app. See [`METHODOLOGY.md`](./METHODOLOGY.md)
for the five ideas it's built on.

## Install

```shell
/plugin marketplace add johngaynor/adlc
/plugin install adlc@adlc
```

Then, inside any project you want to harness:

```shell
/adlc:init
```

`/adlc:init` detects your stack and scaffolds an opinionated `CLAUDE.md` and
a `.claude/lessons.md` — a strong starting point you grow from. It also connects
the repo to Linear, which the lifecycle below requires.

## The ADLC lifecycle

Six stages carry a task from a rough idea to a closed-out record, each its own skill:

```
brainstorm → spec → plan → execute → pr → archive
```

- **`/adlc:brainstorm`** — sharpen a rough idea into a crisp problem statement and
  create the task's Linear issue (`## Idea`, status `Backlog`).
- **`/adlc:spec`** — research and write the specification onto that same issue
  (`## Specification`).
- **`/adlc:plan`** — turn the spec into an ordered, phased plan and a progress
  checklist (`## Plan`, `## Progress`), status `Todo`.
- **`/adlc:execute`** — create the task's branch and git worktree, then work the
  plan phase by phase, ticking `## Progress` live as each phase lands, status
  `In Progress`.
- **`/adlc:pr`** — validate, commit, push, and open the PR; the branch name lets
  Linear auto-link it, and the issue advances to `In Review`.
- **`/adlc:archive`** — once the PR merges, commit a curated summary to
  `docs/adlc/` and close the issue, status `Done`.

**Linear is the live workspace.** One Linear issue is the whole record for a task
while it's in flight — its description holds the canonical `## Idea` /
`## Specification` / `## Plan` / `## Progress` / `## Outcome` sections, upserted in
place as each stage runs. Every lifecycle skill talks to Linear through a single
seam (`reference/pm-seam.md`) instead of calling it ad hoc, so a future PM adapter
could replace Linear without any skill changing.

**The repo gets a curated summary, not a duplicate.** `/adlc:archive` writes a
short `docs/adlc/{date}-{slug}.md` — what shipped, why, how, and any notable
deviations — once the PR merges. It is deliberately not a copy of the full
spec/plan; that detail stays in the (now closed) Linear issue.

Task identity is derived, never a shared pointer: before code exists it's the
Linear issue held in the agent's own session; once `/adlc:execute` creates the
task branch, the branch name (`<initials>/<issue-identifier>-<slug>`) is what
every later stage parses to find the task. Parallel tasks never collide.

> **Requires: Linear MCP.** The harness hard-requires Linear: every lifecycle
> stage refuses to run until `/adlc:init` has connected the repo (a `pm` block in
> `.adlc/config.json`, verified against a live Linear MCP). Set up the Linear MCP
> server first — init walks you through it and is not complete without it.

## Skills

| Skill | What it does | Status |
|-------|--------------|--------|
| `/adlc:init` | Scaffold the harness into the current repo and connect Linear (required) | ✅ implemented |
| `/adlc:brainstorm` | Sharpen an idea into the task's Linear issue (`## Idea`) | ✅ implemented |
| `/adlc:spec` | Research and write the task's specification into Linear (`## Specification`) | ✅ implemented |
| `/adlc:plan` | Write the phased plan and progress checklist into Linear (`## Plan`, `## Progress`) | ✅ implemented |
| `/adlc:execute` | Branch/worktree the task and work the plan phase-by-phase, ticking Linear live | ✅ implemented |
| `/adlc:pr` | Validate, commit, and open the task's PR; advances the Linear task to `In Review` | ✅ implemented |
| `/adlc:archive` | Commit a curated repo summary and close the Linear issue | ✅ implemented |
| `/adlc:add-lesson` | Capture a correction as a durable lesson | ✅ implemented |

## Updating

This plugin uses explicit versioning (`.claude-plugin/plugin.json` → `version`).
Bump it for a release; consumers pull it with:

```shell
/plugin marketplace update adlc
/reload-plugins
```

During active development you can omit `version` so every commit is a new version.

---

## Repo layout

```
adlc/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace listing (this repo is its own marketplace)
├── METHODOLOGY.md           # the spine — the five ideas; every skill quotes this
├── CONVENTIONS.md           # how to author skills in this plugin (the contract)
├── reference/
│   └── pm-seam.md           # the PM operation contract every lifecycle skill consumes
├── templates/               # files /adlc:init and /adlc:archive render into a target repo
│   ├── CLAUDE.md.template
│   ├── lessons.md.template
│   └── archive-summary.md.template
└── skills/
    ├── init/                # the opinionated scaffolder (reference skill)
    ├── brainstorm/          # lifecycle: idea → Linear issue
    ├── spec/                # lifecycle: idea → specification
    ├── plan/                # lifecycle: spec → phased plan + progress checklist
    ├── execute/             # lifecycle: plan → branch/worktree → committed code
    ├── pr/                  # lifecycle: validate → branch → commit → PR
    ├── archive/             # lifecycle: merged PR → curated summary → closed issue
    └── add-lesson/          # self-improvement lessons loop
```

## Contributing / parallel work streams

The original four v0.1 skills (streams A–D below) are implemented, and the six
Linear-lifecycle skills (see "The ADLC lifecycle" above) have since been added
alongside them. When adding or changing a skill, read
[`CONVENTIONS.md`](./CONVENTIONS.md) (skill format) and
[`METHODOLOGY.md`](./METHODOLOGY.md) (vocabulary), and use `skills/init/SKILL.md` as
the worked example.

| Stream | File | Status |
|--------|------|--------|
| **A — Init scaffolder** | `skills/init/SKILL.md` + `templates/` | ✅ implemented (reference skill) |
| **B — Spec workflow** | `skills/spec/SKILL.md` — superseded by the lifecycle; specs live on the Linear issue | ✅ implemented |
| **C — PR workflow** | `skills/pr/SKILL.md` | ✅ implemented |
| **D — Lessons loop** | `skills/add-lesson/SKILL.md` | ✅ implemented |
| **E — Docs & dogfood** | this README + a throwaway test repo | ⏳ in progress |

## License

MIT — see [LICENSE](./LICENSE).
