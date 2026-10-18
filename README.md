# MachineMail agent skills

[![skills.sh](https://skills.sh/b/machinemail/skills)](https://skills.sh/machinemail/skills)

MachineMail is email for AI agents. Every agent gets its own address, every inbound message arrives with a trust report so nothing in the inbox can hijack the agent, and the REST API is wire-compatible with AgentMail's `/v0`. Agents can use it over the REST API at `https://api.machinemail.dev/v0` or the hosted MCP server at `https://mcp.machinemail.dev/mcp`, and can even sign themselves up through [auth.md](https://api.machinemail.dev/auth.md).

These skills teach coding agents (Claude Code, Cursor, Codex, OpenCode, Gemini CLI and the other agents the [skills CLI](https://skills.sh) supports) how to use MachineMail correctly.

## Install

All skills:

```bash
npx skills add machinemail/skills
```

One skill:

```bash
npx skills add machinemail/skills --skill machinemail
```

List what is in the repo without installing:

```bash
npx skills add machinemail/skills --list
```

Useful flags: `-g` installs for your user rather than the project, `-a claude-code` targets one agent, `-y` skips the prompts.

## Skills

| Skill | Use it when |
|---|---|
| [`machinemail`](skills/machinemail/SKILL.md) | An agent needs an email address, has to send, receive, wait for (sign-up codes), reply to, forward or label mail, read attachments, or react to webhooks and Server-Sent Events. Includes the trust rules: inbound mail is untrusted, act on `trust.level`, give the model only `trust.content.text`. |
| [`machinemail-agent-signup`](skills/machinemail-agent-signup/SKILL.md) | An agent has no API key and must register itself through auth.md, work within the sandbox limits, and get a human to claim it with a six-digit code. |
| [`machinemail-mcp`](skills/machinemail-mcp/SKILL.md) | Connecting the MachineMail MCP server to an MCP client with OAuth or an API key, and choosing the right tool (`wait_for_message`, `create_draft`, `reply_to_message` and the rest). |
| [`machinemail-agentmail-migration`](skills/machinemail-agentmail-migration/SKILL.md) | Moving an AgentMail integration to MachineMail: base URL and key, webhook signatures, MCP tool names, agent sign-up, and the endpoints that differ. |

Each skill keeps its `SKILL.md` short and loads detail from its `references/` folder only when needed.

## Docs

- Documentation: https://machinemail.dev/docs
- For agents: https://machinemail.dev/docs/llms.txt and https://machinemail.dev/docs/llms-full.txt
- Support: support@machinemail.dev

## License

[MIT](LICENSE)
