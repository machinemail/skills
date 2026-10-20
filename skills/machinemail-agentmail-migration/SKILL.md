---
name: machinemail-agentmail-migration
description: Moves an existing AgentMail integration to MachineMail, which serves AgentMail's /v0 API. Copies pods, custom domains, inboxes, allow and block lists and webhooks with the `machinemail import agentmail` CLI importer, then switches the base URL to https://api.machinemail.dev and the API key to an mm_live_ key while keeping the agentmail TypeScript or Python SDK. Use when code imports agentmail or AgentMailClient, calls api.agentmail.to, verifies AgentMail webhooks with Svix, uses AgentMail's MCP server, or calls /v0/agent/sign-up, and the user wants to move to MachineMail or compare the two. Covers the importer (dry run, ID map, safe re-runs, DNS cutover, enabling the imported webhooks), what carries over, what differs (error envelope, X-MachineMail-Signature webhooks, auth.md instead of agent sign-up, MCP tool names, hidden labels), endpoints that return 404, and reading MachineMail's trust report through the SDK.
license: MIT
metadata:
  author: MachineMail
  version: "1.1.0"
  homepage: https://machinemail.dev/docs/getting-started/from-agentmail
---

# Switching from AgentMail to MachineMail

MachineMail implements AgentMail's `/v0` API: same paths, field names and enum values. Switching code is a base URL and an API key, and the AgentMail SDKs keep working. The `machinemail import agentmail` CLI command copies the account's settings, so you do not recreate them by hand.

## 1. Get a MachineMail key

Sign up at https://machinemail.dev/signup and copy the API key the dashboard shows (it starts with `mm_live_`). Keys can also be created at https://machinemail.dev/app/api-keys, scoped to the organization, a pod or one inbox, with AgentMail's permission names.

## 2. Copy your settings with the importer

The MachineMail CLI (`machinemail`, also `mm`) reads the AgentMail account and recreates its settings in your MachineMail organization. It never copies mail, and it changes nothing at AgentMail: it only reads.

Install it and sign in:

```bash
curl -fsSL https://machinemail.dev/install.sh | sh   # installs machinemail and mm into ~/.local/bin
machinemail login                                    # or set MACHINEMAIL_API_KEY
```

Public CLI releases are not published yet. If the install script returns 404, tell the user and point them to https://machinemail.dev/docs/cli/installation. Use an organization MachineMail key, or a pod key to import into one pod.

Give the CLI an AgentMail key. It needs only the read permissions `inbox_read`, `pod_read`, `domain_read`, `webhook_read`, `list_entry_read` and `api_key_read`; suggest a short expiry. The CLI never takes the key as an argument:

```bash
export AGENTMAIL_API_KEY=am_us_...                  # or pass --key-file ~/agentmail.key, or paste it at the prompt
machinemail import agentmail --dry-run              # read and plan only: writes nothing, saves no file
machinemail import agentmail                        # asks before creating anything (--yes skips the question)
```

The key stays on the user's machine. The CLI sends it only to AgentMail, never to MachineMail, and shows it masked (`am_us_…a91f`). Do not paste the key into chat or a command line for the user; have them export it or use `--key-file`.

What it copies:

- **Pods**: name and `client_id`. AgentMail's Default Pod maps to the default pod.
- **Custom domains**: added with new DNS records for the user to publish. The CLI does not change DNS.
- **Inboxes**: display name, metadata, `client_id`, status and pod. Custom-domain addresses stay the same. `@agentmail.to` addresses belong to AgentMail and cannot move, so the CLI takes the same name on `machinemail.dev` (with a suffix when that name is taken). Choose another with `--rename old@agentmail.to=new-name`, or leave an inbox out with `--skip old@agentmail.to`.
- **Allow and block lists** at the organization, pod and inbox level.
- **Webhooks**: URL, event types and pod and inbox filters. They are created **disabled**, each with a new signing secret. Enable them after the receivers verify MachineMail signatures (step 4).

What it does not copy: mail (threads, messages, drafts, attachments), API keys (the report lists their names, scopes and permissions to recreate), webhook secrets, custom webhook header values (AgentMail never returns them, and MachineMail does not send custom headers), and read-only list entries that AgentMail manages. The report names everything it skipped.

After a real run the CLI writes an ID map, `agentmail-import-map.json` and `.csv` (`--map-file` changes the path), with each AgentMail id and its MachineMail id. Use it to update stored `pod_id`, `domain_id`, `webhook_id` and `inbox_id` values. It holds the webhook URLs, which can carry tokens, so keep it private. Then the CLI prints a cutover checklist.

Running it again is safe. Every object is created with a `client_id`, so a second run creates nothing new, and after a partial failure it creates only what is missing.

EU keys (`am_eu_`) are read from `api.agentmail.eu`; `--region us|eu` overrides the key prefix. MachineMail stores data in the United States (https://machinemail.dev/privacy), so importing an EU organization moves its settings and future mail out of the EU. Tell the user before running the import, and let them decide.

Still done by hand after the import: publish each domain's DNS records and then switch its MX record (mail keeps going to AgentMail until MX changes; never publish both MX records), enable the webhooks, create MachineMail API keys, and revoke the AgentMail key used for the import.

## 3. Point the client at MachineMail

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

## 4. Replace webhook verification

MachineMail does not sign with Svix. It sends `X-MachineMail-Signature: t=<unix>,v1=<hex>`, the hex HMAC-SHA256 of `<t>.<raw body>` keyed with the webhook `secret`. Replace the Svix check and reject timestamps more than 300 seconds old. Code: https://machinemail.dev/docs/guides/verify-webhooks

Webhooks copied by the importer are disabled and have new secrets. Read each secret with `machinemail webhooks get <webhook-id>`, update the receiver, then turn the webhook on with `machinemail webhooks update <webhook-id> --enabled` (the import checklist prints one command per webhook). Leave off any the report says were disabled at AgentMail. Without the importer, create each webhook again; each gets a new `secret`.

## 5. Update MCP config

Point the agent at `https://mcp.machinemail.dev/mcp` with the MachineMail key as a Bearer token (or OAuth sign-in). Tool names differ from AgentMail's MCP server, so update any prompts or code that name tools; the `machinemail-mcp` skill lists them.

## 6. Replace agent sign-up

`/v0/agent/sign-up`, `/v0/agent/human` and `/v0/agent/verify` are not available. Agents register through https://api.machinemail.dev/auth.md instead; see the `machinemail-agent-signup` skill.

## 7. Fix error handling

`/v0` errors are `{"error":{"type","message","param"?,"code"?}}` with the matching HTTP status. Branch on `error.type` (`validation`, `unauthorized`, `plan_limit_exceeded` (402), `forbidden`, `account_suspended` (403), `not_found`, `conflict`, `rate_limited`, `internal`, `not_implemented`). The `/agent/*` and `/oauth2/*` routes return OAuth-style errors instead: `{error, error_description, message}`. Creates return 201. Messages accept at most 50 recipients.

## 8. Use the trust report

Every inbound message gains a `trust` object. The TypeScript SDK passes it through untouched in snake_case, so read it with a cast:

```ts
const message = await mail.inboxes.messages.get(inboxId, messageId);
const trust = (message as any).trust; // { level, score, content: { text }, reasons, ... }
if (trust?.level === 'quarantined') return;
```

Give the model `trust.content.text`, not `text` or `html`, and decide what the agent may do from `trust.level` (the `machinemail` skill has the policy). Mail that tries to instruct an agent gets the `QUARANTINE` label instead of `INBOX`.

## Checklist

- [ ] `machinemail import agentmail --dry-run` reviewed, then the import run (EU organizations: the user agreed to move data to the US)
- [ ] Each custom domain's DNS records published, then MX switched to MachineMail
- [ ] Stored ids updated from the ID map; correspondents told about any changed addresses
- [ ] Base URL and key changed everywhere (including CI secrets); MachineMail API keys created to match the AgentMail ones
- [ ] Svix verification replaced; imported webhooks enabled after their receivers check `X-MachineMail-Signature`
- [ ] MCP URL and tool names updated
- [ ] Agent sign-up moved to auth.md
- [ ] Error handling reads `error.type`
- [ ] No calls to endpoints that return 404 (see [references/differences.md](references/differences.md))
- [ ] Old AgentMail inboxes kept readable until in-flight threads finish; AgentMail import key revoked

Full comparison, 404 list, and the importer and cutover in detail: [references/differences.md](references/differences.md).
