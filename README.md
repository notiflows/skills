# Notiflows Agent Skills

Agent skills + a hosted MCP for [Notiflows](https://notiflows.com) — create and
manage your notiflows as code, straight from your editor. The skills teach an AI
coding agent the real Notiflows schema and tooling; the bundled MCP server lets
it list, create, validate, publish, and trigger notiflows directly.

## Install

### Claude Code

Installs the skills **and** wires up the Notiflows MCP tools in one step. You're
prompted once for your account token, which is stored in your OS keychain (never
written to a file).

```bash
/plugin marketplace add notiflows/skills
/plugin install notiflows@notiflows-skills
```

Prefer to load it from a clone:

```bash
git clone https://github.com/notiflows/skills
claude --plugin-dir ./skills
```

### Cursor, Codex, and other agents

```bash
npx skills add notiflows/skills
```

The skills are written into your project and pinned in `skills-lock.json`. Set up
the MCP separately when you want the tools too — see the
[MCP docs](https://notiflows.com/docs/ai/mcp).

## The skills

| Skill | What it covers |
|---|---|
| **notiflows-schema** | The `notiflow.json` flat-steps format — step types (trigger/end/channel/wait/digest/throttle/condition), positions, `parent_step_handle`/`branch_handle` branching, condition steps + branches, step gate conditions, and per-type settings (`duration` + `duration_unit`). |
| **notiflows-template-schema** | Channel templates — the seven channel types, the `{channel_type, data}` shape, required `body`/`content_type`, content-type enums per channel, and the `@`-suffix body extraction to `steps/<handle>/body.<ext>`. |
| **notiflows-cli** | The `@notiflows/cli` (bins `notiflows`/`nf`) — install, account-token (`nf_at_`) auth, the full command set, and `nf notiflow run [--draft] -r <id> --data` to trigger notiflows. |
| **notiflows-docs-support** | Where to find authoritative answers — the docs, `llms.txt`, the CLI and MCP server, and how to get unstuck. |

## What you'll need

- A Notiflows account and an **account token** (`nf_at_…`), created in the
  dashboard under **Account → Account tokens**. The Claude Code plugin prompts for
  it; for the CLI or a manual MCP setup, pass it as a bearer token.
- Node.js 18+ (for `npx`).

## Grounding

Every skill describes only fields, commands, and behavior that actually exist in
Notiflows — the API is the source of truth. When a detail is uncertain at runtime,
the skills tell the agent to validate against the API rather than guess.

## License

MIT — see [LICENSE](./LICENSE).
