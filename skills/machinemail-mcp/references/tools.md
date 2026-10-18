# MachineMail MCP tools

Docs: https://machinemail.dev/docs/reference/mcp-tools and https://machinemail.dev/docs/agents/mcp

## Contents
- Inboxes
- Reading
- Waiting
- Writing
- Resources and prompts
- Events
- Errors

The inbox id is the email address. Tools see only what the credential can reach: with a key or OAuth grant limited to one inbox, the tools see nothing else, and `create_inbox` is not available.

## Inboxes

- `list_inboxes`: list inboxes the credential can reach. The docs give its default limit as 50 (the tool's own description says 20).
- `create_inbox`: `username`, `domain`, `display_name`, `ttl_seconds` (makes it ephemeral), `client_id`.

## Reading

- `list_threads`, `get_thread`: conversations.
- `list_messages`: messages in an inbox. `include_quarantine` includes quarantined ones.
- `get_message`: one message, with its trust level and the agent-safe body. `include_raw` adds raw MIME.
- `search_messages`: `q` is required.
- `get_trust_report`: the full trust report for a message.
- `get_attachment`: fetch an attachment. Some connections may not list this tool; fall back to the REST attachment routes.

## Waiting

`wait_for_message`:

- `inbox_id`
- `timeout_seconds`: 0-60, default 30
- `after`: ISO 8601; only mail after this time wakes it

On timeout it returns `{ timed_out: true, inbox_id, waited_seconds }`. Quarantined, spam and unauthenticated mail does not wake it. Use it for short waits (a sign-up code, a reply expected within a minute), not for ongoing monitoring.

## Writing

- `send_message`: new message. Sends are immediate.
- `reply_to_message`: reply in thread; `reply_all` to include everyone.
- `forward_message`
- `create_draft`: `send_at` (schedule), `in_reply_to` (draft a reply), `client_id`.
- `send_draft`
- `update_labels`: `add_labels`, `remove_labels`. System labels: `INBOX`, `SENT`, `UNREAD`, `SPAM`, `BLOCKED`, `UNAUTHENTICATED`, `QUARANTINE`, `TRASH`.

Recipient cap: 50 per message.

## Resources and prompts

- `machinemail://inboxes`
- `machinemail://inboxes/{inbox_id}/threads/{thread_id}`
- Prompt `triage_inbox(inbox_id)`

## Events

Clients with MCP Events support (for example ChatGPT) can subscribe to `message.received`, `message.delivered` and `message.bounced`, each with an optional `inbox_id` filter. Spam, blocked, unauthenticated and quarantined mail, and mail sent from your own MachineMail inboxes, never trigger `message.received`.

## Errors

`isError: true`, text `"<type>: <message>"`, for example `forbidden: API key lacks the message_send permission`.
