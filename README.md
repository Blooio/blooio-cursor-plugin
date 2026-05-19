# Blooio for Cursor

Send iMessages, RCS, and SMS from Cursor. The Blooio plugin connects your Cursor agents and chat to the [Blooio v2 API](https://backend.blooio.com/v2/api/openapi.json) and the hosted [Blooio MCP server](https://blooio.com/integrations/mcp), and bundles skills, rules, and an agent that know the API surface end-to-end.

> Blooio gives you programmatic access to iMessage, RCS, and SMS from a real Apple ID on a real Mac. The plugin makes it trivial to wire that up from Cursor — both for one-off automation and for building production integrations.

---

## What's in the box

- **MCP server** — `mcp.json` registers the hosted Blooio MCP at `https://mcp.blooio.com/v4`. After you set `BLOOIO_API_KEY`, Cursor exposes `me`, `send_message`, `get_message`, `get_message_status`, `cancel_message`, `contact_capabilities`, `webhook_get`, and `webhook_set` as agent tools.
- **Skills** — task-scoped guidance the agent loads on demand:
  - `blooio-mcp-setup` — install and verify the Blooio MCP in Cursor
  - `blooio-api` — auth, base URL, error codes, idempotency, plan limits
  - `blooio-messaging` — send, multipart, attachments, effects, polls, reactions, typing, read receipts
  - `blooio-webhooks` — subscribe, sign, replay, debug delivery
  - `blooio-contacts-groups` — contacts, capabilities, groups, group icons, members
- **Rules** — `blooio-api-conventions` is always-applied guidance the agent follows when it touches Blooio code (E.164, idempotency keys, structured error codes, never logging the API key).
- **Agent** — `blooio-integration-builder` for designing/reviewing a full Blooio integration in your codebase.
- **Command** — `/blooio-quickstart` walks a fresh project from "no key" to "first delivered iMessage" in one prompt.

## Install

### Option 1 — install the plugin (recommended)

1. Open the Cursor Marketplace panel.
2. Search **Blooio** and click **Install**.
3. The MCP server, skills, rules, agent, and command are all activated automatically.

### Option 2 — install just the MCP

If you only want the MCP (not the agent guidance), use the one-click deeplink from the [Blooio MCP integration page](https://blooio.com/integrations/mcp):

```text
cursor://anysphere.cursor-deeplink/mcp/install?name=blooio-mcp&config=eyJ1cmwiOiJodHRwczovL21jcC5ibG9vaW8uY29tL3Y0In0=
```

### Option 3 — clone for local development

```bash
git clone https://github.com/blooio/blooio-cursor-plugin.git ~/.cursor/plugins/local/blooio
```

Then **Developer: Reload Window** in Cursor.

## Configure your API key

The MCP authenticates with a Bearer token. Get one from the [Blooio dashboard](https://app.blooio.com) → **Settings → API Keys**, then export it before launching Cursor:

```bash
export BLOOIO_API_KEY="bl_live_..."
```

`mcp.json` references it as `${BLOOIO_API_KEY}` so the key never lives in your repo.

> If you'd rather scope the key to one workspace, put it in a workspace-level `.env` and start Cursor from a shell that loaded it, or use Cursor's MCP secret UI to override the `Authorization` header for that server.

## Verify the install

Ask the agent: **"Use the Blooio MCP to call `me` and confirm my account works."**

Expected: a JSON blob with your `organization_id`, your devices, and `valid: true`. If you get `401`, your key isn't being passed — recheck the env var.

## Quick examples

Once installed, you can ask the agent things like:

- "Send an iMessage to +15551234567 saying 'deploy succeeded' with a confetti effect."
- "List all webhooks on my Blooio account, then create one pointing at `https://example.com/hooks/blooio` for `message` events."
- "Build me a TypeScript helper that wraps `POST /chats/{chatId}/messages` with idempotency and retries."
- "Inspect the last 20 inbound messages and tell me which contacts haven't replied in 7+ days."

The agent will pick up the relevant skill (`blooio-messaging`, `blooio-webhooks`, etc.) automatically.

## Compatibility

- Cursor IDE (any recent build with MCP support)
- The hosted MCP also works from ChatGPT Agent Builder, Claude Desktop, Cline, and any MCP client implementing the 2024-11-05 spec — see [blooio.com/integrations/mcp](https://blooio.com/integrations/mcp).

## Support

- API & dashboard: [blooio.com](https://blooio.com)
- Docs: [blooio.com/integrations/api](https://blooio.com/integrations/api), [blooio.com/integrations/mcp](https://blooio.com/integrations/mcp)
- Email: [support@blooio.com](mailto:support@blooio.com)
- iMessage: [text us](https://blooio.com/msg/I%20was%20on%20the%20Cursor%20plugin%20and%20have%20a%20question)

## License

MIT — see [LICENSE](./LICENSE).
