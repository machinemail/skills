---
name: machinemail-agent-signup
description: Lets an AI agent sign itself up for MachineMail with no API key and no human present, by following auth.md - register an anonymous identity, get a sandbox inbox and a one-hour access token, then hand off to a human with the six-digit claim code and verification link. Use when an agent needs its own email address right away, when there is no MachineMail API key, when a user says "read machinemail.dev/auth.md", or when an agent must get a human to claim, bind or approve it. Also covers sandbox limits (one inbox, 10 messages a day), token renewal without a refresh token, claim polling errors such as authorization_pending and slow_down, and revoking an agent.
license: MIT
metadata:
  author: MachineMail
  version: "1.0.0"
  homepage: https://machinemail.dev/docs/agents/auth-md
---

# MachineMail agent self-signup (auth.md)

An agent can get a working MachineMail address without a human creating a key first. The source of truth is the live auth.md file, which is written for agents and generated from the server's current configuration:

```bash
curl -s https://api.machinemail.dev/auth.md
```

Read it first and prefer its values (endpoints, `resource`, grant types, limits) over anything below. The flow is: discover, register, claim, exchange, call, revoke.

## 1. Register

```bash
curl -s -X POST https://api.machinemail.dev/agent/identity \
  -H "Content-Type: application/json" \
  -d '{"type":"anonymous","name":"Research assistant for Ada"}'
```

`name` (up to 100 characters) is shown to the human at claim time. The response contains:

- `identity_assertion`: what you exchange for access tokens. Valid for 24 hours.
- `claim_token` (`clm_...`): returned only once. Keep it.
- `pre_claim_scopes`: `inboxes.read`, `messages.read`, `messages.send`, `messages.write`.
- `inbox`: a sandbox address `agent-<12 characters>@machinemail.dev`, `ephemeral: true`, `ttl_seconds: 86400`.

## 2. Exchange for an access token

```bash
curl -s -X POST https://api.machinemail.dev/oauth2/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer \
  --data-urlencode "assertion=$IDENTITY_ASSERTION" \
  --data-urlencode "resource=$RESOURCE"
```

`$RESOURCE` is the root `resource` from `https://api.machinemail.dev/.well-known/oauth-protected-resource` (currently `https://machinemail.dev/`; take it from that response), or leave the parameter out: it is optional, and any other value returns `invalid_target`. The same root token works on both the REST API (`/v0`) and MCP. It lasts 3600 seconds. There is no refresh token: when it expires, exchange the assertion again, until the assertion itself expires after 24 hours.

## 3. Use the inbox

Call the API with `Authorization: Bearer <access_token>`, or connect to `https://mcp.machinemail.dev/mcp` with it. The `machinemail` skill covers sending, waiting and the trust rules. Sandbox limits apply until a human claims the agent; see [references/sandbox-limits.md](references/sandbox-limits.md).

## 4. Get a human to claim the agent

```bash
curl -s -X POST https://api.machinemail.dev/agent/identity/claim \
  -H "Content-Type: application/json" \
  -d '{"claim_token":"clm_...","email":"ada@example.com"}'
```

`email` is optional; with it, only the person signed in with that address can claim the agent. The response has a six-digit `user_code`, a `verification_uri`, `expires_in: 600` and `interval: 5`.

Show the human both, for example: "To claim me, open <verification_uri> and enter code 186569 within 10 minutes." The human can also enter the code at https://machinemail.dev/claim.

Then poll `POST /oauth2/token` every `interval` seconds with the claim grant type listed in auth.md and your `claim_token`. Handle the poll errors:

| `error` | Meaning |
|---|---|
| `authorization_pending` | The human has not finished. Keep polling. |
| `slow_down` | Add 5 seconds to the interval. |
| `expired_token` | The code lapsed: start a new claim. If the 24-hour claim window closed, register again. |
| `access_denied` | The registration was revoked. Stop; register again only if the human still wants it. |

A successful claim returns a token with the post-claim scopes and a new `identity_assertion`, and revokes the pre-claim tokens and the pre-claim assertion (exchanging the old one now returns `invalid_grant`). Switch to the new token at once, and store the new `identity_assertion` and use it for renewals. A claimed agent can keep renewing its token for about a day. For anything longer, ask the human to create an API key at https://machinemail.dev/app/api-keys.

## 5. Revoke

The agent can revoke its own credentials:

```bash
curl -s -X POST https://api.machinemail.dev/oauth2/revoke \
  -d "token=$ACCESS_TOKEN" -d token_type_hint=access_token
```

Posting the identity assertion instead (`token_type_hint=identity_assertion`) revokes the whole registration: every token stops working. A human can list registrations with `GET /v0/agent-registrations?status=pending_claim|claimed|revoked` and remove one with `DELETE /v0/agent-registrations/{agent_id}`.

## Gotchas

- Do not put an auth.md access token in an MCP client config: it expires after an hour and clients do not renew it. Use OAuth sign-in or an API key there.
- Registration is rate limited per IP (1 per hour, 3 per 24 hours). Register once and keep the assertion and claim token.
- Never claim on the human's behalf or guess the code; the human enters it.
- Full step detail and the `service_auth` registration type: [references/auth-flow.md](references/auth-flow.md).
