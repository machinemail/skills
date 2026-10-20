# MachineMail agent skills

[![skills.sh](https://skills.sh/b/machinemail/skills)](https://skills.sh/machinemail/skills)

MachineMail is email for AI agents. Every agent gets its own address, every inbound message arrives with a trust report so the agent can tell what to trust and what to keep away from the model, and the REST API is wire-compatible with AgentMail's `/v0`. Agents can use it over the REST API at `https://api.machinemail.dev/v0` or the hosted MCP server at `https://mcp.machinemail.dev/mcp`, and can even sign themselves up through [auth.md](https://api.machinemail.dev/auth.md).

These skills teach coding agents (Claude Code, Cursor, Codex, OpenCode, Gemini CLI and the other agents the [skills CLI](https://skills.sh) supports) how to use MachineMail correctly.

## Install

Pick skills interactively:

```bash
npx skills add machinemail/skills
```

Every skill:

```bash
npx skills add machinemail/skills --skill '*'
```

Every skill for every supported agent: `npx skills add machinemail/skills --all`.

One skill:

```bash
npx skills add machinemail/skills --skill machinemail
```

A few skills, for example the everyday MCP tasks:

```bash
npx skills add machinemail/skills --skill read-email-safely --skill receive-verification-code --skill triage-inbox --skill reply-and-draft
```

List what is in the repo without installing:

```bash
npx skills add machinemail/skills --list
```

Useful flags: `-g` installs for your user rather than the project, `-a claude-code` targets one agent, `-y` skips the prompts.

## Skills

### Get started

| Skill | Use it when |
|---|---|
| [`machinemail`](skills/machinemail/SKILL.md) | An agent needs an email address, has to send, receive, wait for (sign-up codes), reply to, forward or label mail, read attachments, or react to webhooks and Server-Sent Events. Includes the trust rules: inbound mail is untrusted, act on `trust.level`, give the model only `trust.content.text`. |
| [`machinemail-mcp`](skills/machinemail-mcp/SKILL.md) | Connecting the MachineMail MCP server to an MCP client with OAuth or an API key, and choosing the right tool (`wait_for_message`, `create_draft`, `reply_to_message` and the rest). |
| [`register-agent-inbox`](skills/register-agent-inbox/SKILL.md) | The user wants the agent to set itself up with an inbox: use the MCP tools if they are signed in, otherwise self-register through auth.md and have a person claim the agent. |
| [`machinemail-agent-signup`](skills/machinemail-agent-signup/SKILL.md) | The full auth.md protocol: register with no API key, work within the sandbox limits, poll the claim, renew and revoke tokens. |

### Everyday tasks

| Skill | Use it when |
|---|---|
| [`read-email-safely`](skills/read-email-safely/SKILL.md) | Before reading, summarising or acting on any message, and whenever an email asks the agent to do something: which field to read and what each trust level allows. |
| [`receive-verification-code`](skills/receive-verification-code/SKILL.md) | Signing up for or logging in to another service with a MachineMail address and picking up the code or magic link, after checking it came from that service. |
| [`triage-inbox`](skills/triage-inbox/SKILL.md) | A read-only digest of what is new in an inbox, grouped by trust level, with proposed next steps for the user to approve. |
| [`reply-and-draft`](skills/reply-and-draft/SKILL.md) | Replying, forwarding or composing, and deciding when to save a draft for a person to review instead of sending. |

### Migrate

| Skill | Use it when |
|---|---|
| [`machinemail-agentmail-migration`](skills/machinemail-agentmail-migration/SKILL.md) | Moving an AgentMail integration to MachineMail: copying pods, domains, inboxes, lists and webhooks with `machinemail import agentmail`, then base URL and key, webhook signatures, MCP tool names, agent sign-up, and the endpoints that differ. |

The everyday task skills are written for the MCP tools and point to `machinemail` for the REST API and the full trust reference.

Each skill keeps its `SKILL.md` short; the larger ones load detail from their `references/` folder only when needed.

Want the MCP server connection bundled with the skills as a plugin for Claude Code, Codex, Copilot CLI, Cursor or VS Code? Use [machinemail/agent-plugin](https://github.com/machinemail/agent-plugin).

## Docs

- Documentation: https://machinemail.dev/docs
- For agents: https://machinemail.dev/docs/llms.txt and https://machinemail.dev/docs/llms-full.txt
- Support: support@machinemail.dev

## License

[MIT](LICENSE)
