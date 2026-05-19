---
name: blooio-mcp-setup
description: >-
  Install, configure, and verify the Blooio MCP server in Cursor. Use when the
  user mentions installing Blooio, "connect Cursor to Blooio", "MCP not
  working", "401 from Blooio", missing tools like send_message/me, or anything
  about BLOOIO_API_KEY environment setup.
---

# Blooio MCP setup

The Blooio MCP server is hosted at `https://mcp.blooio.com/v4` (HTTP transport, MCP spec 2024-11-05). Auth is a Bearer token in the `Authorization` header — the same API key you'd use against the REST API at `https://backend.blooio.com/v2/api`.

This plugin ships an `mcp.json` that wires the server up automatically once `BLOOIO_API_KEY` is in the environment Cursor was launched from.

## Tools the MCP exposes

`me`, `contact_capabilities`, `send_message`, `get_message`, `get_message_status`, `cancel_message`, `webhook_get`, `webhook_set`. They mirror the underlying REST endpoints and accept the same fields (E.164 phone numbers, `idempotencyKey`, attachment URLs, etc).

## Setup checklist

1. **Get a key** from [app.blooio.com → Settings → API Keys](https://app.blooio.com). Free trial keys work; they're limited to 20 messages.
2. **Export the env var** in the shell that launches Cursor:
   ```bash
   echo 'export BLOOIO_API_KEY="bl_live_..."' >> ~/.zshrc
   source ~/.zshrc
   ```
   (Use `~/.bash_profile` for bash.)
3. **Quit and relaunch Cursor from that shell** (`open -a Cursor` from Terminal works on macOS). MCP servers inherit env vars at launch — restarting from the dock will reuse the old (empty) environment.
4. **Reload the window**: Cmd+Shift+P → *Developer: Reload Window*.
5. **Verify**: in chat, ask the agent to call the `me` tool. It should return a JSON blob with `valid: true` and your `organization_id`.

## Common failure modes

| Symptom | Cause | Fix |
| --- | --- | --- |
| `401 unauthorized` from any tool | `${BLOOIO_API_KEY}` is empty in Cursor's env | Export the key, relaunch Cursor from a shell that has it, reload window. |
| Tool list is empty | MCP server didn't connect | Open Cursor Settings → MCP. The blooio entry should be green. If red, check that the Authorization header is non-empty. |
| `403 inbound_only_no_prior_inbound` on send | Number is on the Inbound plan; recipient hasn't messaged in first | Inbound numbers are reply-only by design. Either upgrade to Dedicated, or ask the contact to message you first. |
| `429 outbound_limit_reached` | Org-level new-contact cap from Settings tripped | Bump the cap in dashboard, or wait for the rolling window to reset. |
| `429 new_conversation_limit_reached` | Shared plan daily new-conversation cap | Existing conversations still send. Upgrade to Dedicated to lift the cap. |
| `503 No active number available` | None of your numbers are registered/active right now | Check device status in dashboard — usually a Mac that needs to be unlocked or rebooted. |

## When to prefer the REST API over the MCP

Use the MCP when you're driving Blooio from chat or an agent loop — it handles auth and serialization for you. Use the REST API when you're writing application code (a webhook receiver, a CRM integration, a backend service); see the [`blooio-api`](../blooio-api/SKILL.md) skill for that surface.

## Manual install (without this plugin)

If a user is on a Cursor build that doesn't auto-load `mcp.json`, click the install deeplink:

```text
cursor://anysphere.cursor-deeplink/mcp/install?name=blooio-mcp&config=eyJ1cmwiOiJodHRwczovL21jcC5ibG9vaW8uY29tL3Y0In0=
```

…and add the `Authorization: Bearer <key>` header in Cursor's MCP settings. The deeplink intentionally omits the key so it can be copy-pasted publicly.
