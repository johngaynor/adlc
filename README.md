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

`/adlc:init` detects your stack and scaffolds an opinionated `CLAUDE.md`,
a `.claude/lessons.md`, and the project-skills convention (`.ai/skills/` +
discovery symlink, see below) — a strong starting point you grow from: lessons
accumulate, recurring ones become `CLAUDE.md` rules, and whole workflows graduate
into project skills via `/adlc:add-skill`. It can also optionally connect the
repo to Linear, which unlocks the lifecycle below.

## The ADLC lifecycle

Six stages carry a task from a rough idea to a closed-out record, each its own skill:

```
brainstorm → spec → plan → execute → pr → archive
```

- **`/adlc:brainstorm`** — sharpen a rough idea into a crisp problem statement and
  create (or retrofit) the task's Linear issue: a brief 1–2 sentence description
  with key findings in a collapsed `### Notes` block, status `Backlog`.
- **`/adlc:spec`** — research and write the specification onto that same issue
  (`# Specification` — summary paragraph, collapsible topic sections, and a
  Risks & unknowns dropdown).
- **`/adlc:plan`** — turn the spec into an ordered, phased plan and a progress
  checklist (`## Plan`, `## Progress`), status `Todo`.
- **`/adlc:execute`** — create the task's branch and git worktree, then work the
  plan phase by phase, ticking `## Progress` live as each phase lands, status
  `In Progress`.
- **`/adlc:pr`** — validate, commit, push, and open the PR; the branch name lets
  Linear auto-link it, and the issue advances to `In Review`.
- **`/adlc:archive`** — once the PR merges, commit a curated summary to
  `docs/adlc/` and close the issue, status `Done`.

Stages hand off to each other through the workflow bridge, **`/adlc:next`**: each
stage ends by offering *Move to next / Review first / Stop here* (or chains
automatically if you ask for auto mode), so you never have to remember the next
command — and `/adlc:next` on its own resumes a task at whatever comes next.

**Linear is the live workspace.** One Linear issue is the whole record for a task
while it's in flight — its description holds the canonical artifacts (the
brainstorm preamble with its Notes dropdown, the `# Specification` block, and the
`## Plan` / `## Progress` / `## Outcome` sections), upserted in place as each
stage runs. Every lifecycle skill talks to Linear through a single
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

> **Requires: Linear MCP.** `/adlc:brainstorm` through `/adlc:archive` need a
> Linear MCP server configured for the target repo — set it up via `/adlc:init`'s
> optional Linear step. Without it, `/adlc:pr` and `/adlc:add-lesson` still work
> standalone with no PM configured.

## Skills

| Skill | What it does | Status |
|-------|--------------|--------|
| `/adlc:init` | Scaffold the harness into the current repo (incl. the project-skills convention); optionally connect Linear | ✅ implemented |
| `/adlc:brainstorm` | Sharpen an idea into the task's Linear issue (brief description + collapsed Notes) | ✅ implemented |
| `/adlc:spec` | Research and write the task's specification into Linear (`# Specification`) | ✅ implemented |
| `/adlc:plan` | Write the phased plan and progress checklist into Linear (`## Plan`, `## Progress`) | ✅ implemented |
| `/adlc:execute` | Branch/worktree the task and work the plan phase-by-phase, ticking Linear live | ✅ implemented |
| `/adlc:pr` | Validate, branch, commit, and open a PR; advances a resolved Linear task to `In Review` | ✅ implemented |
| `/adlc:archive` | Commit a curated repo summary and close the Linear issue | ✅ implemented |
| `/adlc:add-lesson` | Capture a correction as a durable lesson | ✅ implemented |
| `/adlc:add-skill` | Author a project skill in `.ai/skills/`; graduates skill-sized lessons | ✅ implemented |
| `/adlc:cleanup` | Sweep merged task worktrees (and their branches) and move merged-PR Linear issues to `Done` | ✅ implemented |

## Project skills — extending the harness

A harnessed repo can ship its *own* agent skills alongside the plugin's. The
canonical home is tool-neutral and committed: `.ai/skills/<name>/SKILL.md`,
in the open agent-skills format. `/adlc:init` scaffolds the plumbing: a
committed relative symlink `.claude/skills → ../.ai/skills` (so Claude Code
discovers project skills natively and `/<name>` invokes them), an `AGENTS.md`
pointer for other harnesses, a README seed that keeps the symlink from dangling
on fresh clones, and gitignore rules that keep `.ai/skills/` committed even
where the rest of `.ai/` stays local. See METHODOLOGY.md § "Extending the
harness with project skills" (including the Windows symlink caveat).

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
├── templates/               # files the skills render into a target repo
│   ├── CLAUDE.md.template
│   ├── lessons.md.template
│   ├── skill.md.template    # skeleton /adlc:add-skill renders into .ai/skills/
│   ├── project-skills-README.md.template
│   └── archive-summary.md.template
└── skills/
    ├── init/                # the opinionated scaffolder (reference skill)
    ├── brainstorm/          # lifecycle: idea → Linear issue
    ├── spec/                # lifecycle: idea → specification
    ├── plan/                # lifecycle: spec → phased plan + progress checklist
    ├── execute/             # lifecycle: plan → branch/worktree → committed code
    ├── pr/                  # lifecycle: validate → branch → commit → PR
    ├── archive/             # lifecycle: merged PR → curated summary → closed issue
    ├── add-lesson/          # self-improvement: correction → lesson
    └── add-skill/           # self-improvement: lesson/workflow → project skill
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
