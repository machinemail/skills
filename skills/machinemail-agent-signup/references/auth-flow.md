# auth.md flow in detail

Docs: https://machinemail.dev/docs/agents/auth-md and https://machinemail.dev/docs/agents/claiming. Live file: https://api.machinemail.dev/auth.md (also served at https://machinemail.dev/auth.md).

## Contents
- Discovery
- Registration types
- Token exchange
- Claim ceremony
- Errors
- Revocation

## Discovery

auth.md points at standard OAuth discovery documents:

- `/.well-known/oauth-protected-resource`
- `/.well-known/oauth-authorization-server` (lists `grant_types_supported`, the token endpoint and the claim endpoint)
- `/.well-known/jwks.json`
- For MCP: `https://mcp.machinemail.dev/.well-known/oauth-protected-resource/mcp`

A `401` from `/v0` carries `WWW-Authenticate: Bearer resource_metadata="..."` pointing at the protected-resource document, so an agent that hits the API without credentials can discover the flow from the error alone.

## Registration types

`POST /agent/identity` accepts:

- `{"type":"anonymous","name":"..."}`: returns an `identity_assertion`, a `claim_token`, the pre-claim scopes and a sandbox inbox at once. The human claims later.
- `{"type":"service_auth","login_hint":"person@example.com"}`: returns no assertion and no inbox; it comes with a `claim` block straight away, and only the person signed in as `login_hint` can claim it.

## Token exchange

`POST /oauth2/token`, form-encoded:

- `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`
- `assertion=<identity_assertion>`
- `resource=<resource>`: the value from auth.md; for MCP, `https://mcp.machinemail.dev/mcp`.

Access tokens last 3600 seconds. No refresh token is issued; exchange the assertion again. The assertion expires after 24 hours.

## Claim ceremony

1. `POST /agent/identity/claim` with `{"claim_token":"clm_...","email":"optional"}`. Instead of `claim_token`, the agent may send `{"agent_id":"reg_..."}` with its access token or assertion as the Bearer credential.
2. The response gives `user_code` (6 digits), `verification_uri`, `expires_in: 600`, `interval: 5`.
3. Show the code and link to the human. The claim page at https://machinemail.dev/claim also accepts the code (`?code=<code>` prefills it).
4. Poll `POST /oauth2/token` with the claim grant type from `grant_types_supported` and `claim_token=<clm_...>`, every `interval` seconds.
5. On success, use the new token. The pre-claim tokens are revoked.

## Errors

`/agent/*` and `/oauth2/*` return OAuth-style errors: `{error, error_description, message}`.

Poll errors: `authorization_pending` (keep polling), `slow_down` (add 5 seconds), `expired_token` (start a new claim), `access_denied` (stop).

## Revocation

- Agent side: `POST /oauth2/revoke` with `token=...&token_type_hint=access_token` or `token_type_hint=identity_assertion`.
- Human side: `GET /v0/agent-registrations?status=pending_claim|claimed|revoked` and `DELETE /v0/agent-registrations/{agent_id}`.
