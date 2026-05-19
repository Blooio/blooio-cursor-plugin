---
name: blooio-quickstart
description: Verify the Blooio MCP is connected, pick a number, send a test iMessage, and confirm delivery — end-to-end smoke test.
---

# Blooio quickstart

Run this when a user just installed the Blooio plugin and wants to confirm everything works, or when they say "send me a test message" / "is Blooio connected?".

## Steps

1. **Verify auth.** Call the MCP `me` tool. Expect `valid: true`, an `organization_id`, and a non-empty `devices` array.
   - If 401: `BLOOIO_API_KEY` isn't being passed. Walk through the `blooio-mcp-setup` skill — export the key, relaunch Cursor from a shell that has it, reload window.
   - If `devices` is empty or all `is_active: false`: a Mac is offline/locked. Tell the user; don't try to send.

2. **Pick a from-number.** Call `me/numbers` (or read `me`'s `devices`). If multiple, ask the user which one to send from. Show plan kind (`shared` / `dedicated` / `inbound` / `trial` / `2fa`) — `inbound` is reply-only and won't be able to initiate the test.

3. **Pick a recipient.** Default to the user's own phone (they'll usually have it on the same iCloud account so iMessage works for sure). Otherwise, prompt for an E.164 number.

4. **Check capabilities** with `GET /contacts/{contactId}/capabilities`. Confirm `imessage: true` so the test rides iMessage.

5. **Send the test message.**
   ```http
   POST /chats/{urlEncodedRecipient}/messages
   Idempotency-Key: blooio-quickstart-{timestamp}
   { "text": "👋 Blooio + Cursor are connected. (Quickstart test)", "effect": "balloons" }
   ```

6. **Track delivery.** Poll `GET /chats/{chatId}/messages/{messageId}/status` every ~2s until `status` is `delivered`, `failed`, or 30s elapsed. Report the final state.

7. **Suggest next steps.** Based on what they probably want:
   - "Want to receive replies? Set up a webhook." → load `blooio-webhooks` skill.
   - "Want to send to a group?" → load `blooio-messaging` (multi-recipient section).
   - "Building a CRM-style sync?" → load `blooio-contacts-groups`.

## Failure handling

- `403 inbound_only_no_prior_inbound` — explicitly tell the user this number is reply-only by design. Suggest upgrading to Dedicated for outbound.
- `429 outbound_limit_reached` / `new_conversation_limit_reached` — show the body's `limit` and `current`. Point to dashboard settings.
- `503` — surface the device status; don't retry blindly.

Keep the loop short — five tool calls, max. If anything past step 1 fails, stop and explain rather than spinning.
