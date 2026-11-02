<p align="center">
  <a href="https://json-schema.org/">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/json-schema-org/website/main/public/logo-white.svg">
      <img alt="JSON Schema logo" src="https://raw.githubusercontent.com/json-schema-org/website/main/public/logo-blue.svg" width="120">
    </picture>
  </a>
</p>

<h1 align="center">json-schema-skill</h1>

<p align="center">
  An Agent Skill that teaches AI coding agents to write, validate, fix and migrate JSON Schema
</p>

<p align="center">
  <a href="https://json-schema.org/draft/2020-12"><img alt="JSON Schema 2020-12" src="https://img.shields.io/badge/JSON%20Schema-2020--12-1e5aa8"></a>
  <a href="https://json-schema.org/draft/2019-09"><img alt="JSON Schema 2019-09" src="https://img.shields.io/badge/JSON%20Schema-2019--09-1e5aa8"></a>
  <a href="https://ajv.js.org/"><img alt="Ajv 8" src="https://img.shields.io/badge/Ajv-8-23c8d2"></a>
  <a href="https://python-jsonschema.readthedocs.io/"><img alt="Python jsonschema" src="https://img.shields.io/badge/python--jsonschema-4.x-3776ab?logo=python&logoColor=white"></a>
  <a href="https://skills.sh/"><img alt="Agent Skill" src="https://img.shields.io/badge/Agent-Skill-6f42c1"></a>
</p>

`json-schema-skill` is a portable [Agent Skill](https://skills.sh/) for working with [JSON Schema](https://json-schema.org/). It's plain Markdown (a `SKILL.md` file plus reference docs), and it works with any agent that supports skills, including Claude Code, Codex, Cursor, Gemini CLI, Antigravity, OpenCode and GitHub Copilot. Once installed, your agent loads it automatically when a task involves JSON Schema.

It defaults to **draft 2020-12** and supports **2019-09**, as well as draft-07/06/04 when a file declares them. The agent checks its work with [Ajv](https://ajv.js.org/) or [Python `jsonschema`](https://python-jsonschema.readthedocs.io/), whichever fits your project, installed in a temporary folder rather than your project.

## What it gives your agent

- **Writing schemas** from sample JSON, a description, or existing types, then proving them against valid and invalid examples.
- **Validating data** and turning validator errors into plain explanations. For example, it finds the one real cause behind a wall of `oneOf` errors.
- **Reviewing and fixing schemas**, including bugs validators silently ignore:
  - misspelled keywords (`require`, `minLenght`)
  - `additionalProperties: false` next to `allOf`
  - unanchored `pattern`s
  - overlapping `oneOf` branches
- **Migrating drafts** (draft-04 → 07 → 2019-09 → 2020-12) and reporting which examples change behavior.
- **Converting OpenAPI 3.0 schemas** (`nullable`, boolean `exclusiveMinimum`, `example`) to 2020-12.
- **Bundling** multi-file schemas into one self-contained file.
- **Generating code** (TypeScript, Zod, Pydantic, Go, and others) and flagging what doesn't map cleanly.

It also teaches the agent where validators disagree with the spec and with each other. Some examples:
- Ajv applies keywords next to `$ref` in draft-07.
- Both Ajv's defaults and stock Python `jsonschema` reject `0.3` for `multipleOf: 0.1`.
- `format` is checked only when you opt in.

## Install

### With the `skills` CLI (any agent)

The [`skills` CLI](https://github.com/vercel-labs/skills) detects which agents you have and installs into each one's skills folder:

```bash
npx skills add Suyog241005/json-schema-skill        # current project
npx skills add Suyog241005/json-schema-skill -g     # all projects (user-level)
npx skills add Suyog241005/json-schema-skill -g -a codex cursor   # specific agents only
```

### Manually

Clone the repo and link the folder into your agent's skills directory:

```bash
git clone https://github.com/Suyog241005/json-schema-skill.git
ln -s "$PWD/json-schema-skill" ~/.agents/skills/json-schema      # pick the path for your agent below
```

| Agent | Project-level | User-level |
|---|---|---|
| Codex, Cline, Warp, Zed, Amp | `.agents/skills/` | `~/.agents/skills/` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Cursor | `.agents/skills/` | `~/.cursor/skills/` |
| Gemini CLI | `.agents/skills/` | `~/.gemini/skills/` |
| Antigravity | `.agents/skills/` | `~/.gemini/antigravity/skills/` |
| OpenCode | `.agents/skills/` | `~/.config/opencode/skills/` |
| GitHub Copilot | `.agents/skills/` | `~/.copilot/skills/` |

See the [`skills` CLI agent list](https://github.com/vercel-labs/skills/blob/main/docs/agents.md) for other agents. Linking instead of copying keeps one copy of the skill, so `git pull` updates it for every agent.

### Agents without skill support

Point the agent at the instructions from your project's `AGENTS.md` (or the agent's equivalent rules file):

```markdown
For any JSON Schema work, follow the instructions in path/to/json-schema-skill/SKILL.md.
```

### Requirements

Node.js (for Ajv) or Python 3 (for `jsonschema`) on the machine where the agent runs. The agent installs the validator itself, in a temporary folder.

## Repository layout

```
SKILL.md                      # skill instructions; the description tells agents when to load it
references/                   # loaded on demand, one per task
  keywords.md                 # all 2020-12/2019-09 keywords, gotchas, recipes
  drafts.md                   # draft migration tables and OpenAPI 3.0 → 2020-12 conversion
  ajv.md                      # Ajv in application code
  python-jsonschema.md        # Python jsonschema in application code
  codegen.md                  # schema ⇄ TypeScript / Zod / Pydantic / Go mappings
```

## Acknowledgements

The JSON Schema logo belongs to the [JSON Schema organization](https://github.com/json-schema-org). It is used here only to indicate the specification this skill works with. This project is not affiliated with or endorsed by the JSON Schema organization.
