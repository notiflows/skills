---
name: notiflows-cli
description: The Notiflows CLI (@notiflows/cli, binaries `notiflows` and `nf`) — a notiflows-as-code tool over the Management API. Use to install/authenticate the CLI, scaffold/pull/push/publish/validate/diff notiflows, and trigger runs with `nf notiflow run [--draft] -r <recipient> --data <json>`. Triggers include "install the notiflows cli", "log in with my account token", "push/pull a notiflow", "publish this notiflow", "trigger/test a notiflow", "create a new notiflow", "diff local vs remote", or any command starting with `notiflows`/`nf`.
---

# Notiflows CLI (`@notiflows/cli`)

A "notiflows as code" CLI: pull notiflows into
version-controllable local files, edit them, push them back, and trigger runs.
It is a thin client over the **Management API** (`/api/management/v1`). Built on
OCLIF — commands are space-separated (`nf notiflow push`, NOT `notiflow:push`).

## Start here — install, auth, first push

```bash
npm install -g @notiflows/cli      # binaries: `notiflows` and `nf` (aliases)

nf login                           # paste an account token (prefix nf_at_)
nf whoami --project my-project     # verify auth + project access

nf init                            # create notiflows.json in the repo
nf pull --project my-project       # pull all notiflows into .notiflows/
# ...edit .notiflows/notiflows/<handle>/notiflow.json...
nf notiflow push welcome-series    # push the draft back
nf notiflow publish welcome-series # publish it live
```

## Auth & configuration

- **Account token** (prefix `nf_at_`), stored at
  `~/.config/notiflows/credentials.json` (mode 0600). This is the Management API
  auth (`Authorization: Bearer nf_at_...`).
- **Token resolution precedence**: `--token` flag → `NOTIFLOWS_TOKEN` env →
  stored credentials.
- **Project resolution precedence**: `--project` flag → `NOTIFLOWS_PROJECT` env
  → `notiflows.json` `project` field.
- **Base URL**: defaults to `https://api.notiflows.com/api/management/v1`;
  override with `--base-url` or `NOTIFLOWS_BASE_URL`.
- **`notiflows.json`** (project config, discovered upward from cwd):
  `{ "notiflowsDir": ".notiflows", "project": "my-project" }`. Notiflows live at
  `<notiflowsDir>/notiflows/<handle>/`.

## On-disk layout

```
notiflows.json                      # project config
.notiflows/
  notiflows/
    welcome-series/                 # dir name = notiflow handle (hyphens only)
      notiflow.json                 # the notiflow (see notiflows-schema)
      steps/
        email_1/body.html           # extracted template body (see notiflows-template-schema)
```

## Command set

Top-level:

- `nf login` / `nf logout` — manage the stored account token.
- `nf whoami` — show the authenticated account/token (needs a project).
- `nf init` — scaffold `notiflows.json`.
- `nf pull` — pull ALL notiflows into the working tree (overwrites after one
  confirm).
- `nf push` — push ALL local notiflows. Conflict-aware: warns + skips notiflows
  changed on the server, exits non-zero. (Does not validate.)
- `nf diff` — compare local files vs remote (PUT-body-shape comparison).

`nf notiflow <cmd>` (single-notiflow; handle is inferred from the current
directory if omitted):

- `new <handle>` — scaffold a notiflow. `--name`, `--steps email,wait,sms`
  (comma-separated), `--push` (push after create), `--force`. Always emits
  `trigger` + `end`.
- `pull <handle>` / `pull --all` — pull one (skips existing unless `--force`).
  `--all` PRUNES local dirs missing remotely — destructive, use with care.
- `push <handle>` — upsert the draft, then validate. `--publish` to publish if
  valid, `--force` to override a server conflict.
- `publish <handle>` — publish the current draft live.
- `validate <handle>` — report whether the current version is valid
  (returns `{ valid: boolean }`).
- `diff` — local vs remote for one notiflow.
- `rollback <handle>` — discard unpublished draft changes.
- `activate` / `deactivate <handle>` — toggle `active`.
- `archive <handle>` — soft-delete the notiflow.
- `list` / `get <handle>` / `versions <handle>` — read-only inspection.
- `open <handle>` — open the notiflow in the dashboard (offline, no auth).
- `run <handle>` — trigger a run (see below).

`nf project list` / `nf project get` and `nf channel list` / `nf channel get`
— read-only (projects and channels are configured in the dashboard).

## Triggering runs — `nf notiflow run`

`nf notiflow run` triggers a notiflow, attributed to your account token (shows
as the run's "triggered by" in the dashboard).

```bash
# Run the published version (notiflow must be active AND published):
nf notiflow run welcome-series -r user_123
nf notiflow run welcome-series -r user_123 -r user_456 --data '{"order_id":"ORD-1"}'
nf notiflow run welcome-series --topic order-updates

# Test the current unpublished DRAFT (must be active; need not be published):
nf notiflow run welcome-series --draft -r user_123
```

Flags:

- `-r, --recipient <external_id>` — repeatable recipient external IDs.
- `--topic <name>` — target a topic instead of explicit recipients.
- `-d, --data <json>` — template variables as a JSON object.
- `--actor <external_id>` — the actor performing the action.
- `--draft` — run the current draft (`version: "draft"`) instead of published.

You must provide at least one `--recipient` or a `--topic`.

## Common tasks

**Create a new notiflow:**
```bash
nf notiflow new welcome-series --name "Welcome Series" --steps email,wait,email
# edit .notiflows/notiflows/welcome-series/notiflow.json + steps/*/body.html
nf notiflow push welcome-series
nf notiflow run welcome-series --draft -r test_user   # smoke-test the draft
nf notiflow publish welcome-series
```

**Sync remote → git, edit, push back:**
```bash
nf pull
nf diff                 # see what changed
nf notiflow push <handle>
```

**CI deploy (idempotent):** `nf push` re-pushing unchanged content creates no
new version (server-side SHA idempotency), so it is safe to run on every deploy.

## Gotchas

- **Notiflow handles are hyphens-only** (`welcome-series`) to match Notiflows;
  underscores 422 on push. (Step handles like `email_1` DO allow underscores —
  different rule.)
- An upsert (`push`) sends the FULL flat steps array; omitted steps are removed.
  Always keep `trigger` and `end`.
- Scaffolded template bodies must be non-empty or the push is rejected.
- `push` upserts the draft FIRST, then validates — invalid local state reaches
  the server draft before the warning; validation only gates `--publish`.
- Conflict detection differs by path: single-notiflow `push` is conflict-safe
  via `expected_sha` (409 if the server moved on since you pulled; `--force`
  overrides), while the top-level `push` warns and skips conflicting notiflows.

## Deeper references

- The notiflow.json structure → the **notiflows-schema** skill.
- Channel templates and body files → the **notiflows-template-schema** skill.
- The MCP server (same API, conversational interface) and where to find
  authoritative docs → the **notiflows-docs-support** skill.
