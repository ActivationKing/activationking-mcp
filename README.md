# ActivationKing MCP

Connect your AI assistant to your [ActivationKing](https://activationking.com) account.

ActivationKing runs photo booths, AI experiences and games at live events. This MCP server lets an AI agent (Claude, Cursor, and other MCP clients) read **your own** ActivationKing data, change your **event settings** and **white label branding**, **remote-control your booths**, and manage your **Ops Calendar**: look up events and guest sessions, check credits, adjust a kiosk's screen texts, colors and timings, start, pause or drive a running booth and send it a photo, and create or update companies, event jobs and deals, for example to push a deal won in your CRM straight into Ops as a job.

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
- "On the *Summer Launch* photo booth, set the AI effect screen title to *Pick your style* and turn on surprise mode."
- "Change my white label button color to #E4007C and the app name to *Acme Booth*."
- "Which of my booths are online right now? Pause the one at *Summer Launch*."
- "On the *Summer Launch* booth, start a photo session and send it this picture: https://example.com/photo.jpg"
- "Create this CRM deal as an Ops job: client Acme, photo booth, 14–15 March in Berlin, price 2,400 EUR."
- "Move the Acme job's deal to *Won* and add a line for an extra print station."

### Tools

The authoritative, always up-to-date list with every parameter is in [llms.txt](https://api.activationking.com/llms.txt) and [llms-full.txt](https://api.activationking.com/llms-full.txt). In short:

| Area | Tools |
|---|---|
| Account | `get_account`, `get_credit_balance` |
| Events and sessions | `list_events`, `get_event`, `get_event_stats`, `list_event_sessions`, `get_session` |
| Event settings | `get_settings_catalog`, `get_event_settings`, `update_event_settings` (writes) |
| White label | `get_white_label`, `update_white_label` (writes) |
| Remote control | `list_remote_booths`, `get_remote_booth`, `send_remote_command` (writes), `upload_remote_media` (writes) |
| Ops Calendar (read) | `ops_reference_data`, `ops_list_companies`, `ops_get_company`, `ops_list_jobs`, `ops_get_job`, `ops_get_deal` |
| Ops Calendar (write) | `ops_save_company`, `ops_save_job`, `ops_update_deal`, `ops_delete_company`, `ops_delete_job` |

How the event settings tools work:

- **Catalog first:** `get_settings_catalog` lists the settings an app allows through the API, with type (boolean, integer, number, string, color, enum), default, range or allowed values and a description. The full list is also in [llms-full.txt](https://api.activationking.com/llms-full.txt) under *Event settings catalog*. Currently available for the Photo Booth app (`com.activationking.photobooth`); other apps return `settings_not_supported`.
- **Read:** `get_event_settings` returns an event's current values; `isSet: false` means the kiosk uses the default.
- **Change only what you pass:** `update_event_settings` takes `{ "Key": value }` pairs; `null` resets a key to its default. If any value is invalid (`invalid_setting`), nothing is saved.
- The kiosk picks up the change on its **next sync**.

How the white label tools work:

- `get_white_label` / `update_white_label` take an optional `app` (default `com.activationking.photobooth`). App fields (`Enabled`, `AppName`, `HidePromoSections`) are per app; microsite fields (`MicrositeEnabled`, colors, `TitleText`, `CopyrightText`, `CompanyNameText`) are shared by all your apps.
- **Colors:** a hex string `#RRGGBB` for a solid color, or an object `{color1, color2, color3, fillType: Solid|Gradient, gradientType: Linear|Radial, lineerGradientAngle}` for a gradient. `null` clears a color or text.
- Needs a plan that includes white label (`403 white_label_not_allowed` otherwise). Logos, background media and the custom domain are changed in the app, not through the API.
- Apps pick up the change within about **5 minutes**.

How the remote control tools work:

- **Find the booth:** `list_remote_booths` lists your booths open for remote control (one row per booth and event) with `online`, `operatorEnabled` and the current screen.
- **Read the screen first:** `get_remote_booth` returns what the booth shows right now: `actions` (`role: guest` = an on-screen button, `role: operator` = pause, resume, run…), `choices`, form `fields` with their allowed options, `mediaRequest` when it waits for a photo or video, a `countdown` and the `shareUrl` when there is one.
- **Act on it:** `send_remote_command` runs one action from that screen as the operator and returns the next screen. For a choice use `actionId: "choice"` with `choiceId`; for a form use `actionId: "submitFields"` with `fields`. If the screen changed in the meantime the call is rejected with `stale_state`: read the screen again.
- **Send media:** when the screen has a `mediaRequest` (for example after the `start_upload` action), `upload_remote_media` sends a photo or video by public `https` url or as base64, with its file extension (`.jpg`, `.png`, `.heic`, `.webp`, `.mp4`, `.mov`). Photos up to 50 MB, videos up to 250 MB. The booth uses it like a capture.
- **Requirements:** only the account that runs the booth can control it (organization members can't). Operator control must be on for the event (`403 operator_disabled` otherwise) and the booth must be online (`409 booth_offline`). Operator control is turned on in the booth app, in the event's settings.
- **Online but `booth_timeout`:** if a booth shows `online: true` but every command times out and its screen doesn't change, the booth app is probably connected to a different server or running a version without remote control. Restart it with the current release and try again.
- The REST equivalents are under *Remote control* in the [OpenAPI](https://api.activationking.com/openapi/v1.json) (`/v1/remote/booths`).

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
