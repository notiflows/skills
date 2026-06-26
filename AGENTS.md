# Notiflows — agent guide

This repository packages everything an AI coding agent needs to work with
[Notiflows](https://notiflows.com): notifications modeled as code, as "notiflows".

- **Skills** live in `skills/` (one `SKILL.md` per topic). They cover the notiflow
  schema, channel templates, the `@notiflows/cli`, and where to find authoritative
  answers. Load the one whose `description` matches the task.
- **MCP tools** are wired in `.mcp.json` (hosted at `api.notiflows.com/mcp`). Use
  them to list, create, validate, publish, and trigger notiflows for a project.
  Authentication is a Notiflows account token (`nf_at_…`).

When creating a notiflow, follow the schema skills exactly — handles are
hyphens-only, channel templates need a non-empty body, and an upsert must include
the full flat steps array (including the terminal `trigger`/`end`). If a detail is
uncertain, validate against the API rather than guessing.
