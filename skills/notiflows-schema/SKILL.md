---
name: notiflows-schema
description: The Notiflows notiflow.json schema — the flat-steps format used by the @notiflows/cli and MCP for notiflows-as-code. Use when creating, editing, or validating a notiflow.json file — step types (trigger/end/channel/wait/digest/throttle/condition), step positions and ordering, branching via parent_step_handle/branch_handle, condition steps and branches, per-step gate conditions, and per-type settings (wait/digest/throttle use duration + duration_unit). Triggers include "create a notiflow", "add a step", "add a wait/digest/throttle", "branch on a condition", "edit notiflow.json".
---

# The notiflow.json schema

A notiflow is an ordered, branchable **flat array of steps**. The on-disk
`notiflow.json` maps 1:1 to the Management API `PUT` body. Notiflows is the source
of truth; this file only describes the shape it accepts. There are **no edges** —
ordering and branching come entirely from `position`, `parent_step_handle`, and
`branch_handle`.

## Start here — minimal valid notiflow

```json
{
  "$schema": "https://notiflows.com/schemas/notiflow.json",
  "name": "Welcome Series",
  "active": false,
  "include_in_preferences": false,
  "steps": [
    { "handle": "trigger", "position": 0, "type": "trigger" },
    {
      "handle": "email_1",
      "position": 1,
      "type": "channel",
      "channel_type": "email",
      "template": { "content_type": "html", "subject": "Welcome!", "body@": "steps/email_1/body.html" }
    },
    { "handle": "end", "position": 2, "type": "end" }
  ]
}
```

Rules that always hold:

- **Every notiflow has `trigger` (position 0) and `end` (last position).
  NEVER drop them.** They are auto-seeded by Notiflows and by `nf notiflow new`.
- **`steps` is a flat array** (no nesting, no edges). Order = `position`
  (integer, unique within a scope).
- **`handle` is identity and immutable.** It is the stable per-step ID used for
  cross-references. Step handles match `^[a-z_]+(_\d+)?$` (underscores allowed,
  e.g. `email_1`). NOTE: the **notiflow** handle (the directory name) is
  hyphens-only, no underscores — that is a different rule.
- Top-level fields: `name` (string), `active` (bool), `include_in_preferences`
  (bool), `steps` (array). `$schema` and a `__meta` block are written on pull
  and stripped before push (read-only).

## Step types

`type` is one of: `trigger`, `end`, `channel`, `wait`, `digest`, `throttle`,
`condition`.

| type | purpose | key fields |
|---|---|---|
| `trigger` | entry point (position 0) | — |
| `end` | terminal step (last position) | — |
| `channel` | deliver a notification | `channel_type`, `channel_handle`, `template` (see notiflows-template-schema) |
| `wait` | delay the flow | `settings: { duration, duration_unit }` |
| `digest` | batch notifications over a window | `settings: { duration, duration_unit, digest_key? }` |
| `throttle` | rate-limit | `settings: { duration, duration_unit, throttle_key? }` |
| `condition` | route into branches | `settings: { branches: [...] }` |

`trigger`, `end`, and `channel` steps take no `settings` block of their own
(channel carries a `template` instead).

## Function step settings — duration + duration_unit

`wait`, `digest`, and `throttle` all use the same two timing fields. The field
names are **`duration` (integer) + `duration_unit`** — NOT `unit`.

- `duration_unit` ∈ `seconds | minutes | hours | days`.
- `wait`: `{ "duration": 1, "duration_unit": "days" }`
- `digest`: `{ "duration": 1, "duration_unit": "minutes", "digest_key": "order_id" }`
  — `digest_key` (optional) groups notifications by a property from the trigger
  `data`. The window stays open for the duration, then flushes one digest.
- `throttle`: `{ "duration": 1, "duration_unit": "minutes", "throttle_key": "user_id" }`
  — `throttle_key` (optional) scopes the rate limit.

## Condition steps and branches (routing)

A `condition` step forks the flow. Its `settings.branches` is an array; each
branch is taken when its `conditions` match. Children of the branch are normal
steps that carry `parent_step_handle` (the condition step's handle) +
`branch_handle` (the branch's handle).

```json
{
  "handle": "condition_1",
  "position": 1,
  "type": "condition",
  "settings": {
    "branches": [
      {
        "handle": "branch_1",
        "name": "High priority",
        "finish": false,
        "conditions": [
          {
            "operator": "and",
            "conditions": [
              { "property": "priority", "operator": "equal_to", "value": "high", "value_type": "static" }
            ]
          }
        ]
      }
    ]
  }
}
```

Then a child step under that branch:

```json
{
  "handle": "sms_1",
  "position": 0,
  "type": "channel",
  "channel_type": "sms",
  "parent_step_handle": "condition_1",
  "branch_handle": "branch_1",
  "template": { "content_type": "plaintext", "body@": "steps/sms_1/body.txt" }
}
```

Branch rules:

- A condition step needs **>= 2 branches** to be valid (and <= 10).
- Each branch: `handle` (required), `name` (required), `finish` (bool — stops
  the flow after this branch completes), `conditions` (array of condition
  groups). At least one branch must continue (not all `finish: true`).
- `position` on branch children is scoped to the branch (starts at 0 within
  the branch).
- Nesting depth of condition steps is capped at 5.

## Step gate conditions (vs condition steps)

These are SEPARATE concepts:

- **Condition step** (`type: condition`, above) = routing fork into branches.
- **Step gate conditions** = a per-step `conditions` field on ANY step that
  decides whether THAT step runs. Same condition-group shape as a branch's
  `conditions`.

```json
{
  "handle": "email_1",
  "position": 1,
  "type": "channel",
  "channel_type": "email",
  "conditions": [
    {
      "operator": "and",
      "conditions": [
        { "property": "user.opted_in", "operator": "equal_to", "value": "true", "value_type": "static" }
      ]
    }
  ],
  "template": { "content_type": "html", "subject": "Hi", "body@": "steps/email_1/body.html" }
}
```

## Condition group + condition shape

A `conditions` value (on a branch or as a step gate) is an array of **condition
groups**. Each group: `operator` (`and` | `or`) + `conditions` (array of
individual conditions).

Each individual condition:

- `property` (string, required) — the data path, e.g. `order.total`.
- `operator` (required) — one of: `equal_to`, `not_equal_to`, `greater_than`,
  `less_than`, `greater_than_or_equal_to`, `less_than_or_equal_to`, `contains`,
  `does_not_contain`, `starts_with`, `ends_with`, `is_empty`, `is_not_empty`.
- `value` (string) — the comparison value (omit for `is_empty`/`is_not_empty`).
- `value_type` — `static` (literal) or `ref` (reference another property);
  defaults to `static`.

## Optional/extra fields

- `name`, `description` on a step are harmless local labels — Notiflows does not
  persist them.
- `provider_settings` on a channel step round-trips (per-step provider
  overrides).
- On pull, the CLI writes a read-only `__meta` block (`version`,
  `published_version`, `status`, `has_unpublished_changes`, `sha`,
  `created_by`, `published_by`) and a `$schema`. Both are stripped before push;
  do not write them by hand.

## Deeper references

- Channel `template` shape and the `@`-suffix body extraction → the
  **notiflows-template-schema** skill.
- Pushing/pulling/validating/running notiflows → the **notiflows-cli** skill.
- Authoritative validation rules live in Notiflows; when in doubt, push and read
  the validation error rather than guessing.
