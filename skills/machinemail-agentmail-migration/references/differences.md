# AgentMail vs MachineMail in detail

Source: https://machinemail.dev/docs/getting-started/from-agentmail and https://machinemail.dev/docs/reference/agentmail-compatibility (lists every operation).

## Contents
- What carries over
- What MachineMail adds
- What is different
- Not available yet (404)
- Importing settings
- Lists after an import
- Cutover by hand

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

## Importing settings

`machinemail import agentmail` (the MachineMail CLI, also `mm`) copies an AgentMail account's settings into a MachineMail organization. It runs on the user's machine, reads AgentMail with the user's AgentMail key, and writes through the public API with the MachineMail key. The AgentMail key is sent only to AgentMail, and nothing at AgentMail changes. Docs: https://machinemail.dev/docs/getting-started/from-agentmail#import-your-settings and https://machinemail.dev/docs/cli/commands/import.

1. Install the CLI (`curl -fsSL https://machinemail.dev/install.sh | sh`; public releases are not published yet, see https://machinemail.dev/docs/cli/installation) and run `machinemail login`. Use an organization key, or a pod key to import into one pod.
2. Create an AgentMail key with `inbox_read`, `pod_read`, `domain_read`, `webhook_read`, `list_entry_read` and `api_key_read`, and a short expiry. Give it through `AGENTMAIL_API_KEY`, `--key-file <path|->` or the hidden prompt; it is never accepted as an argument. A pod-scoped key imports only that pod, an inbox-scoped key only that inbox.
3. `machinemail import agentmail --dry-run` prints what the key can see and the plan, and writes nothing.
4. `machinemail import agentmail` asks, then creates. It writes the ID map (`agentmail-import-map.json` and `.csv`, or `--map-file`) and prints the cutover checklist.

Region: `am_us_` keys read `api.agentmail.to`, `am_eu_` keys read `api.agentmail.eu`; `--region us|eu` overrides. MachineMail stores data in the United States, so an EU organization's settings and future mail leave the EU. The CLI warns about this; make sure the user has decided.

Re-runs are safe: each object carries a `client_id` (AgentMail's own, or `am:<agentmail id>`), so a second run creates nothing, and after a partial failure it fills the gaps. An address that already exists in the organization and was not imported is never taken over; use `--rename` or `--skip`.

| AgentMail | On MachineMail |
|---|---|
| Pods | Name and `client_id`. The Default Pod maps to the default pod. |
| Custom domains | Added with new DNS records to publish. The CLI does not touch DNS. |
| Inboxes | Display name, metadata, `client_id`, status and pod. Custom-domain addresses keep their address. |
| `@agentmail.to` inboxes | Same name on `machinemail.dev`, a suffixed name when taken or reserved, `--rename old=new` to choose, `--skip` to leave out. |
| Allow and block lists | Every entry at organization, pod and inbox level. Read-only entries AgentMail manages are not imported. |
| Webhooks | URL, event types, pod and inbox filters, `client_id`. Created disabled with a new secret. A webhook filtered on both pods and inboxes becomes two. `message.opened`, `message.security.*` and `calendar.event.*` are dropped. |

Imported inboxes are created with `import_source: "agentmail"`, so their `inbox.created` events are logged but not sent to webhooks.

Not imported: mail (threads, messages, drafts, attachments), API keys (listed with name, scope and permissions to recreate), webhook secrets, custom header values (MachineMail sends no custom headers; the report lists the header names), polling webhooks, read-only list entries, and settings with no equivalent such as Agent Armor mode and open tracking.

The command checks plan limits before writing anything and stops when the import does not fit (custom domains beyond the Free plan, or more than 1,000 `machinemail.dev` inboxes). Exit codes: 0 done, 1 plan limit or some items failed, 2 usage error, 3 key refused or not allowed to import, 5 or 6 AgentMail rate limiting or unreachable, 130 interrupted.

## Lists after an import

- AgentMail uses the narrowest level with a match. MachineMail combines organization, pod and inbox entries: an allow match beats a block match, and any allow entry makes that direction allow-only.
- At most 500 entries per direction across the organization, all levels and list types together.
- Receive lists also apply to replies. Reply lists are stored but not applied yet.
- Mail from a blocked sender is stored with the `BLOCKED` label and withheld from agents, not rejected.

## Cutover by hand

The importer does not do these:

1. **DNS, per custom domain.** Publish the DKIM, MAIL FROM, SPF and DMARC records the CLI printed (merge into an existing SPF record; keep an existing DMARC record). Lower the MX TTL, then replace AgentMail's MX with MachineMail's in one change. Never publish both. Mail keeps going to AgentMail until MX changes. Then wait for `VERIFIED` (`machinemail domains get <domain-id>`). Guide: https://machinemail.dev/docs/guides/custom-domain
2. **Stored ids.** Pods, domains, webhooks and renamed inboxes have new ids; update code and config from the ID map. Old `@agentmail.to` addresses keep receiving at AgentMail and nothing forwards them, so tell correspondents.
3. **Webhooks.** Switch receivers to `X-MachineMail-Signature`, read each new secret with `machinemail webhooks get <webhook-id>`, then `machinemail webhooks update <webhook-id> --enabled`. Leave off any that were disabled at AgentMail.
4. **API keys.** Create MachineMail keys to match the ones the report lists, and change the base URL in code.
5. **Mail history** stays at AgentMail. Keep that account readable until in-flight threads finish, then revoke the AgentMail key used for the import.
