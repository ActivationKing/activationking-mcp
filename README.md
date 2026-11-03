# ActivationKing MCP

Connect your AI assistant to your [ActivationKing](https://activationking.com) account.

ActivationKing runs photo booths, AI experiences and games at live events. This MCP server lets an AI agent (Claude, Cursor, and other MCP clients) read **your own** ActivationKing data and manage your **Ops Calendar**: look up events and guest sessions, check credits, and create or update companies, event jobs and deals, for example to push a deal won in your CRM straight into Ops as a job.

| | |
|---|---|
| Endpoint | `https://api.activationking.com/mcp` |
| Transport | MCP Streamable HTTP (stateless) |
| Auth | `Authorization: Bearer ak_live_...` (personal API key) |
| Rate limit | 300 requests per minute per key |
| Reference | [llms.txt](https://api.activationking.com/llms.txt) · [llms-full.txt](https://api.activationking.com/llms-full.txt) · [OpenAPI](https://api.activationking.com/openapi/v1.json) · [Interactive docs](https://api.activationking.com/docs) |

This repository holds the connection guide, a Claude Code plugin and the MCP Registry manifest. The server itself is hosted by ActivationKing; there is nothing to install or run.

## 1. Get an API key

1. Sign in at [activationking.com/dashboard/api-keys](https://activationking.com/dashboard/api-keys).
2. Create a key. It starts with `ak_live_` and is shown once.
3. Store it as an environment variable rather than pasting it into config files:

```bash
# macOS / Linux
export ACTIVATIONKING_API_KEY="ak_live_..."
```

```powershell
# Windows (persists for new terminals)
setx ACTIVATIONKING_API_KEY "ak_live_..."
```

A key only sees the data of the user who created it. Ops Calendar tools additionally follow that user's role in each organization.

## 2. Connect your client

### Claude Code: plugin (recommended)

The plugin adds the server and a skill that teaches Claude how to use the tools safely.

```text
/plugin marketplace add activationking/activationking-mcp
/plugin install activationking@activationking
```

Make sure `ACTIVATIONKING_API_KEY` is set in the environment Claude Code starts from.

### Claude Code: server only

```bash
claude mcp add --transport http activationking https://api.activationking.com/mcp \
  --header "Authorization: Bearer ${ACTIVATIONKING_API_KEY}"
```

### Cursor

`~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

```json
{
  "mcpServers": {
    "activationking": {
      "url": "https://api.activationking.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ACTIVATIONKING_API_KEY}"
      }
    }
  }
}
```

### Claude Desktop

Claude Desktop's config file starts local servers, so use the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) bridge (requires Node.js). Edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "activationking": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://api.activationking.com/mcp",
        "--header", "Authorization:${AUTH_HEADER}"
      ],
      "env": {
        "AUTH_HEADER": "Bearer ak_live_..."
      }
    }
  }
}
```

### Other MCP clients

Any client that supports Streamable HTTP with a custom header works: point it at `https://api.activationking.com/mcp` and send `Authorization: Bearer <your key>`.

> **Web connectors (claude.ai, ChatGPT):** these add remote servers through an OAuth sign-in. ActivationKing MCP currently authenticates with API keys only, so web connectors cannot connect yet. Use one of the clients above.

## 3. What you can do

Try prompts like:

- "List my events from last month and show the session count for each."
- "How many prints and AI credits did the event *Summer Launch* use?"
- "What's my AI credit balance?"
- "Create this CRM deal as an Ops job: client Acme, photo booth, 14–15 March in Berlin, price 2,400 EUR."
- "Move the Acme job's deal to *Won* and add a line for an extra print station."

### Tools

The authoritative, always up-to-date list with every parameter is in [llms.txt](https://api.activationking.com/llms.txt) and [llms-full.txt](https://api.activationking.com/llms-full.txt). In short:

| Area | Tools |
|---|---|
| Account | `get_account`, `get_credit_balance` |
| Events and sessions | `list_events`, `get_event`, `get_event_stats`, `list_event_sessions`, `get_session` |
| Ops Calendar (read) | `ops_reference_data`, `ops_list_companies`, `ops_get_company`, `ops_list_jobs`, `ops_get_job`, `ops_get_deal` |
| Ops Calendar (write) | `ops_save_company`, `ops_save_job`, `ops_update_deal`, `ops_delete_company`, `ops_delete_job` |

How the Ops tools work:

- **Organization:** Ops tools take an `organizationId`. Call `get_account` to see the organizations you belong to.
- **Reference values:** phases, event and venue types, member ids, products and deal statuses come from `ops_reference_data`.
- **Sync from your CRM without duplicates:** pass `externalSource` + `externalId` (your system's id) to `ops_save_company` / `ops_save_job`. The first call creates the record; later calls with the same pair update it.
- **Partial updates:** only the fields you pass change.
- **New jobs open a deal automatically;** use `ops_update_deal` to set its status, currency and price lines.
- **Permissions** follow your role in the organization (for example, deals need Deal.View / Deal.Edit). The organization needs an Ops Calendar subscription; without one, Ops tools return an error.
- **Write tools are annotated** (`readOnlyHint` / `destructiveHint`), so MCP clients ask for confirmation before calling them.

## Security

- Treat your API key like a password. Never commit it, paste it into shared configs or put it in a prompt.
- Prefer environment variables (as in the examples above).
- If a key leaks, revoke it at [activationking.com/dashboard/api-keys](https://activationking.com/dashboard/api-keys) and create a new one.
- Review write actions (create, update, delete) before approving them in your client.

## Support

- API reference: [api.activationking.com/docs](https://api.activationking.com/docs)
- Issues with this repository: open a GitHub issue.
- Account or data questions: contact ActivationKing support via [activationking.com](https://activationking.com).

## License

[MIT](LICENSE)
