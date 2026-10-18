# Events, webhooks and streaming

Docs: https://machinemail.dev/docs/concepts/webhooks, https://machinemail.dev/docs/guides/verify-webhooks, https://machinemail.dev/docs/guides/realtime-events

## Contents
- Webhooks
- Verifying a delivery
- Server-Sent Events
- WebSocket
- Event types

## Webhooks

Create one:

```bash
curl -s https://api.machinemail.dev/v0/webhooks \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/hooks/mail","event_types":["message.received"],"inbox_ids":["support-agent@machinemail.dev"]}'
```

The response includes a `secret` (`whsec_...`). Store it; it signs every delivery. Send a test delivery with `POST /v0/webhooks/{webhook_id}/test`.

Delivery headers:

- `X-MachineMail-Signature: t=<timestamp>,v1=<signature>`
- `X-MachineMail-Event`
- `X-MachineMail-Event-Id`
- `X-MachineMail-Delivery-Id`

## Verifying a delivery

`v1` is the hex HMAC-SHA256, keyed with the webhook secret, of `<t>.<raw request body>` (`t` is Unix seconds). Compute it over the raw request bytes (before JSON parsing), compare in constant time, reject timestamps more than 300 seconds from your clock to stop replays, and deduplicate retries by `X-MachineMail-Event-Id`.

```python
import hmac, hashlib, time

def verify(secret: str, header: str, body: bytes, tolerance: int = 300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    if abs(time.time() - int(parts["t"])) > tolerance:
        return False
    expected = hmac.new(secret.encode(), f"{parts['t']}.".encode() + body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, parts["v1"])
```

A webhook says that something happened; fetch the message with the API and apply the trust rules before acting on it.

## Server-Sent Events

```bash
curl -N "https://api.machinemail.dev/v0/events/stream?inbox_id=support-agent@machinemail.dev&event_types=message.received" \
  -H "Authorization: Bearer $TOKEN"
```

- Needs the `metrics_read` permission.
- Sends a `: ping` comment every 15 seconds.
- Reconnect with `Last-Event-ID` to resume (up to 500 events back).
- `?token=` is accepted instead of the header on this route and on `/v0/ws` only, for clients that cannot set headers.

## WebSocket

`/v0/ws` carries the same events.

## Event types

`message.received`, `message.received.spam`, `message.received.quarantined`, `message.received.unauthenticated`, `message.received.blocked`, `message.sent`, `message.delivered`, `message.bounced`, `message.complained`, `message.rejected`, `draft.sent`, `inbox.created`, `inbox.expired`, `domain.updated`, `domain.verified`, `webhook.test`.

Subscribe agents to `message.received` so spam, quarantined, unauthenticated and blocked mail does not wake them.
