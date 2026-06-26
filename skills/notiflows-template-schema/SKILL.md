---
name: notiflows-template-schema
description: The Notiflows channel template schema for channel steps in notiflow.json — the {channel_type, data} template shape, per-channel fields, required body/content_type, content_type enums per channel, and the @-suffix convention that extracts the body to a separate file under steps/<handle>/. Use when creating or editing a channel step's template (email/sms/in_app/mobile_push/web_push/chat/webhook), choosing a content_type, or working with extracted body files (body.html/body.md/body.txt/body.json). Triggers include "write an email template", "set up an SMS/push/in-app/webhook message", "edit the body file", "what content_type for...".
---

# Notiflows channel templates

A `channel` step in a notiflow.json carries a **template** = the content
delivered on that channel. On the wire (Management API) a template is
`{ "channel_type": "<type>", "data": { ...fields } }`. In the on-disk
notiflow.json the CLI flattens it: the step has a `channel_type` field and a
`template` object holding just the `data` fields (the CLI re-wraps it on push).

## Start here — channel types and what's required

`channel_type` ∈ `email | sms | in_app | mobile_push | web_push | chat |
webhook`.

Almost every channel requires a non-empty **`body`** and a **`content_type`**.
An empty/blank body makes the notiflow un-pushable (Notiflows validates `body`
as required).

| channel_type | required fields | content_type values (default) | body file ext |
|---|---|---|---|
| `email` | `subject`, `body`, `content_type` | `visual`, `html`, `plaintext` (`plaintext`) | `.html` |
| `sms` | `body`, `content_type` | `plaintext` (`plaintext`) | `.txt` |
| `in_app` | `body`, `content_type` | `markdown` (`markdown`) | `.md` |
| `chat` | `body`, `content_type` | `markdown`, `json` (`markdown`) | `.md` |
| `web_push` | `title`, `body`, `content_type` | `plaintext` (`plaintext`) | `.txt` |
| `mobile_push` | `title`, `body`, `content_type` | `plaintext` (`plaintext`) | `.txt` |
| `webhook` | `content_type` (body optional) | `json`, `plaintext` (`json`) | `.json` |

## The @-suffix body extraction convention

To keep templates editable and git-friendly, the **main body field is extracted
to a file** at `steps/<step_handle>/body.<ext>` and referenced inline by a
pointer key: the field name with an `@` suffix.

In notiflow.json:

```json
"template": {
  "content_type": "html",
  "subject": "Welcome aboard",
  "body@": "steps/email_1/body.html"
}
```

with `steps/email_1/body.html`:

```html
<p>Hi {{ user.name }}, welcome!</p>
```

- The CLI reads the file, replaces `body@` with the real `body` field, and wraps
  the whole thing as `{ channel_type, data }` before pushing.
- Only the `body` field is extracted (one content file per step). Everything
  else (`subject`, `title`, `content_type`, etc.) stays inline.
- You can also inline the body directly as `"body": "..."` instead of using
  `body@` — both push the same. The `@`-pointer is the default for editability.
- Template bodies support Liquid templating (`{{ var }}`) resolved at runtime
  against the trigger `data` / recipient.

## Per-channel field reference

### email (`.html` body)
- `subject` (required), `body` (required), `content_type` (`visual` | `html` |
  `plaintext`), `raw_body` (optional, the source for visual editing).

### sms (`.txt` body)
- `body` (required), `content_type` (`plaintext`).

### in_app (`.md` body)
- `body` (required), `content_type` (`markdown`).
- Optional: `action_url`, `action_type` (`default` | `single` | `multi`),
  `primary_action` (map), `secondary_action` (map).

### chat (`.md` body)
- `body` (required), `content_type` (`markdown` | `json`), `raw_body`
  (optional).

### web_push (`.txt` body)
- `title` (required), `body` (required), `content_type` (`plaintext`).
- Optional: `icon`, `image`, `action_url`.

### mobile_push (`.txt` body)
- `title` (required), `body` (required), `content_type` (`plaintext`).
- Optional: `subtitle`, `image_url`, `action_url`, `sound`, `badge_count`
  (int >= 0), `category`, `thread_id`, `collapse_id`, `interruption_level`
  (`passive` | `active` | `time_sensitive` | `critical`), `priority` (`normal` |
  `high`), `relevance_score` (float 0.0–1.0), `ttl` (int >= 0),
  `android_channel_id`.

### webhook (`.json` body)
- `content_type` (`json` | `plaintext`), `url`, `method` (`GET` | `POST` |
  `PUT` | `PATCH` | `DELETE`, default `POST`), `headers` (map, default `{}`),
  `body` (optional — webhook body is not strictly required).

## Examples

Email step (in notiflow.json):

```json
{
  "handle": "email_1",
  "position": 1,
  "type": "channel",
  "channel_type": "email",
  "channel_handle": "transactional-email",
  "template": { "content_type": "html", "subject": "Your receipt", "body@": "steps/email_1/body.html" }
}
```

In-app step with an action:

```json
{
  "handle": "in_app_1",
  "position": 2,
  "type": "channel",
  "channel_type": "in_app",
  "template": {
    "content_type": "markdown",
    "action_type": "single",
    "action_url": "https://app.example.com/orders/{{ order_id }}",
    "body@": "steps/in_app_1/body.md"
  }
}
```

Webhook step:

```json
{
  "handle": "webhook_1",
  "position": 3,
  "type": "channel",
  "channel_type": "webhook",
  "template": {
    "content_type": "json",
    "url": "https://hooks.example.com/notify",
    "method": "POST",
    "headers": {},
    "body@": "steps/webhook_1/body.json"
  }
}
```

## Notes

- `channel_handle` ties the step to a configured delivery channel (channels are
  configured in the dashboard; the CLI/MCP can only read them). A channel step
  needs a channel to publish, but you can create the template first.
- Steps with no template yet pull without a body file — that is valid (not data
  loss), it just isn't deliverable until you add content + a channel.

## Deeper references

- The surrounding step/branch/condition structure → the
  **notiflows-schema** skill.
- Pulling/pushing templates and the body files → the **notiflows-cli** skill.
