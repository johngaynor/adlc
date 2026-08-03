# ADLC — AI Development Life Cycle

A portable engineering harness for AI coding agents, packaged as a **Claude Code
plugin**. Install it into any repo to get an opinionated `CLAUDE.md` (with a Task
Router and boundary-labeled rules), plus skills for spec-first development, PR
discipline, and a self-improvement lessons loop.

The methodology is deliberately framework-agnostic — it works on a TypeScript
monorepo, a Python service, or a mobile app. See [`METHODOLOGY.md`](./METHODOLOGY.md)
for the four ideas it's built on.

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
a `.claude/lessons.md`, and a `.ai/specs/` directory — a strong starting point you
grow from.

## Skills

| Skill | What it does | Status |
|-------|--------------|--------|
| `/adlc:init` | Scaffold the harness into the current repo | ✅ implemented |
| `/adlc:write-spec` | Write a phased spec before non-trivial work | 🚧 stub (Stream B) |
| `/adlc:ship-pr` | Validate, branch, commit, and open a PR | 🚧 stub (Stream C) |
| `/adlc:add-lesson` | Capture a correction as a durable lesson | 🚧 stub (Stream D) |

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
├── METHODOLOGY.md           # the spine — the four ideas; every skill quotes this
├── CONVENTIONS.md           # how to author skills in this plugin (the contract)
├── templates/               # files /adlc:init renders into a target repo
│   ├── CLAUDE.md.template
│   ├── lessons.md.template
│   └── spec.md.template
└── skills/
    ├── init/                # ✅ the opinionated scaffolder
    ├── write-spec/          # 🚧 stub
    ├── ship-pr/             # 🚧 stub
    └── add-lesson/          # 🚧 stub
```

## Contributing / parallel work streams

The shared **contract** (installable shell + methodology spine + skill conventions)
is done, so the remaining skills are independent and can be built in parallel. Each
is one `SKILL.md` file with a build contract already written inside it.

| Stream | File | Contract |
|--------|------|----------|
| **A — Init scaffolder** | `skills/init/SKILL.md` + `templates/` | ✅ done (reference implementation — copy its shape) |
| **B — Spec workflow** | `skills/write-spec/SKILL.md` | see the stub |
| **C — PR workflow** | `skills/ship-pr/SKILL.md` | see the stub |
| **D — Lessons loop** | `skills/add-lesson/SKILL.md` | see the stub |
| **E — Docs & dogfood** | this README + a throwaway test repo | run `/adlc:init` end-to-end, fix rough edges |

Before filling a stub, read [`CONVENTIONS.md`](./CONVENTIONS.md) (skill format) and
[`METHODOLOGY.md`](./METHODOLOGY.md) (vocabulary). Use `skills/init/SKILL.md` as the
worked example.

## License

MIT — see [LICENSE](./LICENSE).
