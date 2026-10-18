# Limits and errors

Docs: https://machinemail.dev/docs/reference/limits, https://machinemail.dev/docs/reference/rate-limits, https://machinemail.dev/docs/reference/errors. Pricing as Markdown: https://machinemail.dev/pricing.md

## Plans

| Plan | Allowance |
|---|---|
| Free | 3,000 messages a month and 100 a day, 1 GB, 1 custom domain |
| Pro | 7,500 inbound, 2,500 outbound |
| Team | 45k inbound, 15k outbound |
| Scale | 200k inbound, 65k outbound |

Going over returns `402 plan_limit_exceeded` with `limit_key`, `resets_at` and `Retry-After`. Wait or tell the human; do not retry in a loop.

## Per-message and per-request limits

- 50 recipients per message (5 in an unclaimed sandbox).
- At most 100 attachments and 10 MB per message; 16 MB request body.
- Page size defaults to 50, max 500.
- 1,000 inboxes per organization on the shared domain.
- `wait_for_message` waits at most 60 seconds per call.

## Rate limits

Most of the API has no request rate limit, and there are no `X-RateLimit-*` headers. The limited areas are sign-in, agent registration, claims (10 per 10 minutes) and sandbox sends. Sandbox numbers are in the `machinemail-agent-signup` skill.

## Error envelope

```json
{"error":{"type":"forbidden","message":"API key lacks the message_send permission"}}
```

`param` and `code` are present when relevant.

| `type` | Status | What to do |
|---|---|---|
| `validation` | 400 | Fix the request; `param` names the field. |
| `unauthorized` | 401 | Token missing or expired. An auth.md token lasts one hour: exchange the assertion again. |
| `plan_limit_exceeded` | 402 | Wait until `resets_at` or ask the human to upgrade. |
| `forbidden` | 403 | The key lacks a permission; ask for one that has it. |
| `account_suspended` | 403 | Stop and tell the human. |
| `not_found` | 404 | Check the inbox id (it is the full address) and ids. |
| `conflict` | 409 | E.g. inbox address taken; pick another username. |
| `rate_limited` | 429 | Honour `Retry-After`. |
| `internal` | 500 | Retry later. |
| `not_implemented` | 501 | The feature is not available. |

The `/agent/*` and `/oauth2/*` routes use OAuth-style errors instead: `{error, error_description, message}`.
