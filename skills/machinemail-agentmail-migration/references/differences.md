# AgentMail vs MachineMail in detail

Source: https://machinemail.dev/docs/getting-started/from-agentmail and https://machinemail.dev/docs/reference/agentmail-compatibility (lists every operation).

## Contents
- What carries over
- What MachineMail adds
- What is different
- Not available yet (404)
- Moving existing mail

## What carries over

Same paths and fields for:

- Inboxes: create, list, get, update, delete, search. `inbox_id` is the email address.
- Messages: list, get, send, reply, reply-all, forward, update labels, delete, raw, attachments, `batch-get`, `batch-update`, search.
- Threads at `/v0/threads` and `/v0/inboxes/{inbox_id}/threads`.
- Drafts, including `send_at` scheduling.
- Webhooks: create, list, get, update, delete (different signature header).
- Domains: create, list, get, update, delete, verify, `zone-file`.
- Lists at `/v0/lists/{direction}/{type}`, also under pods and inboxes.
- Pods and pod-nested routes under `/v0/pods/{pod_id}/...`.
- API keys with AgentMail permission names at org, pod and inbox level.
- `GET /v0/organizations`, `GET /v0/auth/me` (plus usage counters).
- Metrics: `/v0/metrics/events`, `/usage`, `/rates`, with MachineMail event types.
- WebSocket at `/v0/ws` with `subscribe` messages.
- `client_id` idempotency on creates.

## What MachineMail adds

AgentMail clients ignore unknown fields, so these do not break anything.

- `trust` on every inbound message, `trust_level` on list items, and `GET /v0/inboxes/{inbox_id}/messages/{message_id}/trust`.
- A `QUARANTINE` label and a `message.received.quarantined` event. Lists hide quarantined mail unless you pass `include_quarantine=true`.
- Ephemeral inboxes (`ttl_seconds`, `ephemeral`, `expires_at`) and an `inbox.expired` event.
- Catch-all domains (`catch_all`, `catch_all_ttl_seconds`, `inbound_enabled`), DNS host detection and Domain Connect.
- Webhook delivery log: `GET /v0/webhooks/{webhook_id}/deliveries`, replay and test.
- Event log and SSE stream: `GET /v0/events`, `GET /v0/events/stream`.
- Suppressions at `/v0/suppressions`, filled from bounces and complaints.
- Export: `GET /v0/inboxes/{inbox_id}/export?format=mbox|jsonl`.
- Agent self-registration through auth.md.

## What is different

| Area | Difference |
|---|---|
| Error bodies | `{"error": {"type", "message", "param"}}` with the matching HTTP status. |
| Webhook signatures | `X-MachineMail-Signature: t=<unix>,v1=<hex>`, HMAC-SHA256 of `<t>.<body>` with the webhook `secret`. |
| Webhook and stream bodies | AgentMail's shape (`type: "event"`, `event_type`, `event_id`, and the `message`, `thread` or `send` objects at the top level) plus a `data` object with the same payload and a `timestamp`. |
| Event types | Adds `message.received.quarantined`, `inbox.created`, `inbox.expired`, `draft.sent`, `webhook.test`. No `message.opened`, no calendar events. |
| Hidden labels | Lists hide `SPAM`, `BLOCKED`, `TRASH`, `UNAUTHENTICATED` and `QUARANTINE` unless you pass the matching `include_*` flag. |
| Permissions | AgentMail's names, plus `inbound_raw` for `POST /inbound/raw`. |
| Agent sign-up | `/v0/agent/sign-up`, `/v0/agent/human`, `/v0/agent/verify` are not available; use auth.md. |
| Inbox event log | `GET /v0/inboxes/{inbox_id}/events` returns MachineMail's event log for that inbox, not AgentMail's label-event log. |
| Creates | Return 201. |
| Recipients | 50 per message. |
| MCP | Different tool names. |

## Not available yet (404)

- Apps and accounts (`/v0/apps`, `/v0/accounts`, and the inbox and pod variants)
- Calendar (`/v0/inboxes/{inbox_id}/calendar/...`)
- Custom webhook headers (`.../webhooks/{webhook_id}/headers`)
- Hosted domain setup links (`/v0/domains/{domain_id}/setup-link`)
- `POST /v0/inboxes/{inbox_id}/authorize`
- Attachment downloads through threads and drafts. Fetch attachments through the message instead.

## Moving existing mail

There is no importer. AgentMail data, and addresses on AgentMail's own domain, stay with AgentMail.

1. Create the new inboxes on MachineMail, on `machinemail.dev` or your own domain.
2. For a custom domain, add it to MachineMail and switch its MX, SPF and DKIM records to the ones MachineMail gives you (https://machinemail.dev/docs/guides/custom-domain).
3. Recreate webhooks; each gets a new `secret`.
4. Keep the old AgentMail inboxes readable until in-flight threads are finished.
