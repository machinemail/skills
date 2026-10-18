---
name: machinemail-agentmail-migration
description: Switches an existing AgentMail integration to MachineMail, which serves AgentMail's /v0 API, by changing the base URL to https://api.machinemail.dev and the API key to an mm_live_ key while keeping the agentmail TypeScript or Python SDK. Use when code imports agentmail or AgentMailClient, calls api.agentmail.to, verifies AgentMail webhooks with Svix, uses AgentMail's MCP server, or calls /v0/agent/sign-up, and the user wants to move to MachineMail or compare the two. Covers what carries over, what differs (error envelope, X-MachineMail-Signature webhooks, auth.md instead of agent sign-up, MCP tool names, hidden labels), endpoints that return 404, reading MachineMail's trust report through the SDK, and moving mail and domains.
license: MIT
metadata:
  author: MachineMail
  version: "1.0.0"
  homepage: https://machinemail.dev/docs/getting-started/from-agentmail
---

# Switching from AgentMail to MachineMail

MachineMail implements AgentMail's `/v0` API: same paths, field names and enum values. Switching is a base URL and an API key. The AgentMail SDKs keep working.

## 1. Get a MachineMail key

Sign up at https://machinemail.dev/signup and copy the API key the dashboard shows (it starts with `mm_live_`). Keys can also be created at https://machinemail.dev/app/api-keys, scoped to the organization, a pod or one inbox, with AgentMail's permission names.

## 2. Point the client at MachineMail

TypeScript:

```ts
import { AgentMailClient } from 'agentmail';

const mail = new AgentMailClient({
  apiKey: process.env.MACHINEMAIL_API_KEY,
  baseUrl: 'https://api.machinemail.dev', // the only change
});
```

Python:

```python
import os
from agentmail import AgentMail

mail = AgentMail(
    api_key=os.environ["MACHINEMAIL_API_KEY"],
    base_url="https://api.machinemail.dev",  # the only change
)
```

HTTP: replace `https://api.agentmail.to/v0` with `https://api.machinemail.dev/v0` and send the MachineMail key as `Authorization: Bearer`.

The official `agentmail` npm package is tested against MachineMail (inboxes, messages, threads, drafts, webhooks, domains). The Python SDK calls the same paths but is not part of that test suite, so test the calls you use.

## 3. Replace webhook verification

MachineMail does not sign with Svix. It sends `X-MachineMail-Signature: t=<unix>,v1=<hex>`, the hex HMAC-SHA256 of `<t>.<raw body>` keyed with the webhook `secret`. Replace the Svix check, reject timestamps more than 300 seconds old, and recreate each webhook (each gets a new `secret`). Code: https://machinemail.dev/docs/guides/verify-webhooks

## 4. Update MCP config

Point the agent at `https://mcp.machinemail.dev/mcp` with the MachineMail key as a Bearer token (or OAuth sign-in). Tool names differ from AgentMail's MCP server, so update any prompts or code that name tools; the `machinemail-mcp` skill lists them.

## 5. Replace agent sign-up

`/v0/agent/sign-up`, `/v0/agent/human` and `/v0/agent/verify` are not available. Agents register through https://api.machinemail.dev/auth.md instead; see the `machinemail-agent-signup` skill.

## 6. Fix error handling

`/v0` errors are `{"error":{"type","message","param"?,"code"?}}` with the matching HTTP status. Branch on `error.type` (`validation`, `unauthorized`, `plan_limit_exceeded` (402), `forbidden`, `account_suspended` (403), `not_found`, `conflict`, `rate_limited`, `internal`, `not_implemented`). The `/agent/*` and `/oauth2/*` routes return OAuth-style errors instead: `{error, error_description, message}`. Creates return 201. Messages accept at most 50 recipients.

## 7. Use the trust report

Every inbound message gains a `trust` object. The TypeScript SDK passes it through untouched in snake_case, so read it with a cast:

```ts
const message = await mail.inboxes.messages.get(inboxId, messageId);
const trust = (message as any).trust; // { level, score, content: { text }, reasons, ... }
if (trust?.level === 'quarantined') return;
```

Give the model `trust.content.text`, not `text` or `html`, and decide what the agent may do from `trust.level` (the `machinemail` skill has the policy). Mail that tries to instruct an agent gets the `QUARANTINE` label instead of `INBOX`.

## Checklist

- [ ] Base URL and key changed everywhere (including CI secrets)
- [ ] Svix verification replaced; webhooks recreated with new secrets
- [ ] MCP URL and tool names updated
- [ ] Agent sign-up moved to auth.md
- [ ] Error handling reads `error.type`
- [ ] No calls to endpoints that return 404 (see [references/differences.md](references/differences.md))
- [ ] Old AgentMail inboxes kept readable until in-flight threads finish

Full comparison, 404 list and how to move mail and domains: [references/differences.md](references/differences.md).
