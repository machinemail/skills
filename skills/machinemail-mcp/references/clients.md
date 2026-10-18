# MCP client setup

Source: https://machinemail.dev/docs/agents/clients. Every client uses `https://mcp.machinemail.dev/mcp`. OAuth first; the API-key variant reads the key from `MACHINEMAIL_API_KEY` so it stays out of committed files. Apps connected with OAuth appear under Settings, Connected apps in the dashboard, where they can be disconnected.

## Contents
- Claude Code
- claude.ai and Claude Desktop
- ChatGPT
- Cursor
- VS Code
- OpenCode
- Codex
- Gemini CLI
- Zed
- Windsurf
- Goose
- JetBrains IDEs

## Claude Code

```bash
claude mcp add --transport http --scope user machinemail https://mcp.machinemail.dev/mcp
```

Run `/mcp`, select `machinemail`, choose Authenticate. `--scope project` writes `.mcp.json` for the team (no secret in it with OAuth):

```json
{ "mcpServers": { "machinemail": { "type": "http", "url": "https://mcp.machinemail.dev/mcp" } } }
```

API key: add `--header "Authorization: Bearer $MACHINEMAIL_API_KEY"`, or in `.mcp.json` use `"headers": { "Authorization": "Bearer ${MACHINEMAIL_API_KEY}" }`. Check with `claude mcp list`.

## claude.ai and Claude Desktop

Settings, Connectors, Add custom connector. Name `MachineMail`, URL `https://mcp.machinemail.dev/mcp`, leave the advanced OAuth settings empty, then Add and Connect.

API key in Claude Desktop, through `mcp-remote` (needs Node.js), in `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "machinemail": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.machinemail.dev/mcp", "--header", "Authorization:${MACHINEMAIL_AUTH}"],
      "env": { "MACHINEMAIL_AUTH": "Bearer mm_live_..." }
    }
  }
}
```

## ChatGPT

Add a plugin with `https://mcp.machinemail.dev/mcp` as the server URL, sign in with OAuth leaving client ID and secret empty, approve in the tab that opens. ChatGPT supports MachineMail's MCP Events: have a long-running agent subscribe to `message.received` for its inbox instead of polling.

## Cursor

`~/.cursor/mcp.json` (or `.cursor/mcp.json`):

```json
{ "mcpServers": { "machinemail": { "url": "https://mcp.machinemail.dev/mcp" } } }
```

Log in under Settings, MCP. API key: `"headers": { "Authorization": "Bearer ${env:MACHINEMAIL_API_KEY}" }`.

## VS Code

`.vscode/mcp.json`:

```json
{ "servers": { "machinemail": { "type": "http", "url": "https://mcp.machinemail.dev/mcp" } } }
```

Start it and allow authentication. API key: add an `inputs` entry of type `promptString` with `"password": true` and `"headers": { "Authorization": "Bearer ${input:machinemail-api-key}" }`.

## OpenCode

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": { "machinemail": { "type": "remote", "url": "https://mcp.machinemail.dev/mcp", "enabled": true } }
}
```

Sign in with `opencode mcp auth machinemail`. API key: set `"oauth": false` and `"headers": { "Authorization": "Bearer {env:MACHINEMAIL_API_KEY}" }`. Check with `opencode mcp list`.

## Codex

`~/.codex/config.toml`:

```toml
[mcp_servers.machinemail]
url = "https://mcp.machinemail.dev/mcp"
bearer_token_env_var = "MACHINEMAIL_API_KEY"
```

Or leave out `bearer_token_env_var` and run `codex mcp login machinemail`. Check with `codex mcp list`.

## Gemini CLI

`~/.gemini/settings.json`:

```json
{ "mcpServers": { "machinemail": { "httpUrl": "https://mcp.machinemail.dev/mcp" } } }
```

Sign in with `/mcp auth machinemail`. API key: `"headers": { "Authorization": "Bearer $MACHINEMAIL_API_KEY" }`.

## Zed

```json
{ "context_servers": { "machinemail": { "url": "https://mcp.machinemail.dev/mcp" } } }
```

## Windsurf

`~/.codeium/windsurf/mcp_config.json`:

```json
{ "mcpServers": { "machinemail": { "serverUrl": "https://mcp.machinemail.dev/mcp" } } }
```

API key: `"headers": { "Authorization": "Bearer ${env:MACHINEMAIL_API_KEY}" }`.

## Goose

`goose configure`, Add Extension, Remote Extension (Streamable HTTP), name `machinemail`, URL `https://mcp.machinemail.dev/mcp`.

## JetBrains IDEs

AI Assistant supports static headers only: use an API key, or run `mcp-remote` (stdio), which signs in with OAuth:

```json
{ "mcpServers": { "machinemail": { "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.machinemail.dev/mcp"] } } }
```

JetBrains Air and the GitHub Copilot plugin sign in with OAuth directly from the URL.
