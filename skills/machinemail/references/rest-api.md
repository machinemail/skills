# MachineMail REST API reference

Base URL: `https://api.machinemail.dev/v0`. Auth: `Authorization: Bearer <token>` (API key `mm_live_...` or an auth.md access token). Full generated reference: https://machinemail.dev/docs/reference/api

## Contents
- Inboxes
- Messages
- Threads
- Drafts
- Attachments
- Labels
- Test injection
- Permissions an API key can carry

## Inboxes

| Call | Notes |
|---|---|
| `POST /v0/inboxes` | Body `{username, display_name, domain?, ttl_seconds?}`. Returns 201 with `inbox_id` (the address, e.g. `support-agent@machinemail.dev`). 409 if the address is taken. |
| `GET /v0/inboxes/availability?username=<name>` | Check an address before creating it. |

An inbox created with `ttl_seconds` is ephemeral: it is deleted with its mail once it expires, and each new inbound message pushes the expiry out again. Good for one-off sign-ups; do not use an ephemeral address for an account that must be recoverable, because the address can be taken by someone else after it expires.

The shared `machinemail.dev` domain allows 1,000 inboxes per organization.

## Messages

| Call | Notes |
|---|---|
| `POST /v0/inboxes/{inbox_id}/messages/send` | Body `{to[], subject, text, html?, cc, bcc, reply_to, attachments, headers}`. Returns 201 `{message_id, thread_id, timestamp}`. |
| `GET /v0/inboxes/{inbox_id}/messages?labels=INBOX&after=<ISO 8601>` | List messages. Page size defaults to 50, max 500. |
| `GET /v0/inboxes/{inbox_id}/messages/{message_id}` | Full message, including `trust`. |
| `GET /v0/inboxes/{inbox_id}/messages/{message_id}/trust` | The trust report on its own. |
| `PATCH /v0/inboxes/{inbox_id}/messages/{message_id}` | Change labels, e.g. `{"remove_labels":["UNREAD"]}` to mark read. |
| `POST /v0/inboxes/{inbox_id}/messages/{message_id}/reply` | Reply in thread, e.g. `{"text":"Got it, thanks."}`. |
| `POST /v0/inboxes/{inbox_id}/messages/{message_id}/reply-all` | Reply to everyone on the message. |
| `POST /v0/inboxes/{inbox_id}/messages/{message_id}/forward` | Forward the message. |

## Threads

- `GET /v0/threads` lists threads across the organization.
- `GET /v0/inboxes/{inbox_id}/threads` lists one inbox's threads.

## Drafts

- `/v0/inboxes/{inbox_id}/drafts` creates and lists drafts.
- `POST /v0/inboxes/{inbox_id}/drafts/{draft_id}/send` sends one.

Use a draft whenever the message you are answering is below `trust.level` `verified`, or whenever a human should approve the text first. Drafts can also be scheduled (`send_at` on the MCP `create_draft` tool).

## Attachments

- `GET /v0/inboxes/{inbox_id}/messages/{message_id}/attachments/{attachment_id}` returns attachment metadata.
- `.../attachments/{attachment_id}/download` returns the bytes.

Limits: at most 100 attachments and 10 MB per message, 16 MB request body. Attachment content is untrusted like the body; see [trust.md](trust.md).

## Labels

System labels: `INBOX`, `SENT`, `UNREAD`, `SPAM`, `BLOCKED`, `UNAUTHENTICATED`, `QUARANTINE`, `TRASH`. Filter lists with `?labels=`. Add or remove labels with `add_labels` / `remove_labels`. List calls hide `SPAM`, `BLOCKED`, `TRASH`, `UNAUTHENTICATED` and `QUARANTINE` unless you pass the matching `include_*` flag (for example `include_quarantine=true`).

## Test injection

`POST /inbound/raw` with `Content-Type: message/rfc822` and an `X-Recipient` header delivers a raw RFC 5322 message to an inbox, for tests. It needs the `inbound_raw` permission, which is never implied and must be granted by name. Injected mail is never `verified` or `known`.

## Permissions an API key can carry

Keys can be scoped to the organization (`POST /v0/api-keys`), a pod (`POST /v0/pods/{pod_id}/api-keys`) or one inbox (`POST /v0/inboxes/{inbox_id}/api-keys`), and limited to permissions such as `inbox_read`, `inbox_create`, `message_read`, `message_send`, `message_reply`, `message_update`, `thread_read`, `draft_create`, `draft_send`, `webhook_*` and `metrics_read`. A missing permission returns `403 forbidden` naming it.
