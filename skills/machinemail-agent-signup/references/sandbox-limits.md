# Sandbox limits (unclaimed agents)

Source: https://machinemail.dev/docs/agents/auth-md and https://machinemail.dev/docs/reference/rate-limits. auth.md is generated from the live configuration, so its numbers win if they differ.

## The sandbox

- One inbox, `agent-<12 characters>@machinemail.dev`.
- The inbox lives 24 hours; each inbound message extends it.
- The agent has 24 hours to be claimed.

## Sending

- 10 messages per rolling 24 hours.
- 10 distinct recipients per rolling 24 hours.
- At most 5 recipients per message.
- Over the message or recipient budget: `429 rate_limited` with `Retry-After`.
- Too many recipients on one message: `400 validation`.
- Sending to a disposable address: `403 disposable_email_not_allowed`.

## Registration

- 1 per hour and 3 per 24 hours per IPv4 address or IPv6 /64.
- 10 per 24 hours per IPv6 /48.
- About 500 unclaimed sandboxes can be live across the whole service, so registration can fail at busy times even within the per-IP limits.

## Claims

Claim starts are rate limited (10 per 10 minutes).

## After the claim

The agent works under the human's organization and plan, with the post-claim scopes. It can keep renewing its token for about a day; for longer-running work the human creates an API key.
