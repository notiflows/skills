---
name: notiflows-docs-support
description: Where to find authoritative Notiflows answers and how to get unstuck — the docs, the live llms.txt, which Notiflows API and auth to use for a given job, the CLI and MCP, and how to resolve schema or validation questions against the source of truth. Use when unsure which API or auth to use, which tool to reach for (CLI vs MCP vs SDK), where the docs and API reference live, or when a push or validation question needs an authoritative answer rather than a guess.
---

# Notiflows docs & support

How to find authoritative answers about Notiflows and how to get unstuck. The
**Notiflows API is the single source of truth**; the CLI, MCP, SDKs, and docs all
consume it and never redefine it.

## Start here — find the answer

1. **Product & conceptual docs + API reference**: the Notiflows docs at
   <https://notiflows.com/docs>. They also serve **`llms.txt`** (live) — a
   machine-readable index designed for AI agents to discover and pull
   authoritative content. Reach for `llms.txt` first when an agent needs grounding.
2. **Exact schema / validation rules**: the API is authoritative. The API
   reference (generated from the OpenAPI specs) reflects the real contract — never
   hand-write a schema from memory.
3. **When a field/rule is unclear**: prefer pushing and reading the validation
   error over guessing. The Management API surfaces field-level validation errors.

## Which API to use

Notiflows has a few APIs; pick by what you're doing (they differ by auth and consumer):

| API | Auth | Use it for |
|---|---|---|
| **Management** (`/api/management/v1`) | `Authorization: Bearer nf_at_...` (account token) | notiflows-as-code — the CLI, MCP, and AI agents |
| **Admin** (`/api/admin/v1`) | `x-notiflows-api-key` + `x-notiflows-secret-key` | triggering notifications from your product backend; backend SDKs |
| **User** (`/api/user/v1`) | `pk_` api-key + per-user JWT | in-app feed / preferences in a client app (the JS SDKs) |

Rules of thumb:

- **Creating/managing notiflows as an agent or in CI → Management API** (the CLI
  and MCP both use this; account token `nf_at_`).
- **Triggering notifications from your product backend → Admin API**, or for
  agent/CI test runs use the Management API's run endpoint via `nf notiflow run`.
- **In-app feed / preferences in a client app → User API** (`@notiflows/client` +
  `@notiflows/react`).

## AI developer surfaces (CLI + MCP)

Two interfaces over the **same Management API**, differing only in how you drive them:

- **`@notiflows/cli`** (binaries `notiflows` / `nf`) — notiflows-as-code:
  pull/edit/push/publish/validate/diff and `nf notiflow run`. Best for scripting,
  CI, and git-tracked notiflows. See the **notiflows-cli** skill.
- **Notiflows MCP** — a hosted MCP server at `https://api.notiflows.com/mcp` for AI
  editors (Claude Code / Cursor / Claude Desktop). Manage notiflows
  conversationally. Connect with the `mcp-remote` bridge, passing your account
  token as a bearer header:
  `npx -y mcp-remote https://api.notiflows.com/mcp --header "Authorization: Bearer nf_at_…"`.

MCP tool surface (one connection spans every project on the account — pass an
optional `project` slug per call; use `list_projects` to discover slugs):

- Identity: `whoami`, `list_projects`.
- Notiflows: `list_notiflows`, `get_notiflow`, `upsert_notiflow`,
  `validate_notiflow`, `publish_notiflow`, `activate_notiflow`, `rollback_notiflow`.
- Trigger: `run_notiflow` (send notifications; published version by default,
  `draft: true` to test the current draft; attributed to the account token).
- Versions: `list_versions`, `get_version`.
- Templates: `get_template`, `upsert_template`.
- Channels: `list_channels`, `get_channel` (read-only).
- Docs: `search_docs`.

What the MCP server deliberately **does NOT** do: no channel writes (channels are
configured in the dashboard), no template validation tool, and no delete/archive
(no destructive tools).

The agent toolkit (`@notiflows/agent-toolkit`) exposes each published notiflow as a
typed, callable tool for AI agents (Vercel AI SDK / OpenAI / LangChain). See the
`@notiflows/agent-toolkit` package README.

## How to get unstuck

- **"Which auth/key do I use?"** → match the API table above to what you're doing.
- **"Is this notiflow.json valid?"** → `nf notiflow validate <handle>`, or push and
  read the field errors. See the **notiflows-schema** and **notiflows-template-schema**
  skills for the shape.
- **"My push lost all my steps"** → an invalid step can cause Notiflows to reject
  the batch; ensure every step (incl. `trigger`/`end`) is valid and every channel
  template `body` is non-empty.
- **"Conceptual / product question"** → the docs + `llms.txt`.
- **"Exact API request/response shape"** → the generated API reference, not memory.

## Deeper references

- The notiflow.json shape → **notiflows-schema**.
- Channel templates → **notiflows-template-schema**.
- Driving the CLI → **notiflows-cli**.
