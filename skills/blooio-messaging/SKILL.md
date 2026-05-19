---
name: blooio-messaging
description: >-
  Send and track Blooio messages — text, attachments, multipart, link previews,
  iMessage send-with-effect, native polls, reactions (tapbacks + emoji),
  typing indicators, read receipts, message cancellation, and per-message
  delivery status. Use when the user wants to send, draft, schedule, react to,
  or check the status of a message via Blooio.
---

# Blooio messaging

The single most-used endpoint is `POST /chats/{chatId}/messages`. Everything in this skill is built around it. Always:

- URL-encode the `chatId` (`+15551234567` → `%2B15551234567`).
- Set `Idempotency-Key: <uuid>` on every send. Networks fail; retries are the default.
- Treat `202 Accepted` as **queued, not delivered**. Read the next section to track delivery.

## Lifecycle

| State | Meaning |
| --- | --- |
| `queued` | Accepted by Blooio, not yet handed to the device. |
| `pending` | On the device, waiting for the iMessage/SMS network. |
| `sent` | Left the device. Most senders treat this as "success". |
| `delivered` | Confirmed delivery on recipient's device. |
| `read` | Recipient has read receipts on and opened it. (Not guaranteed.) |
| `failed` | Permanent failure. Body has `error_code` + `error_message`. |
| `cancellation_requested` / `cancelled` | You called `DELETE` and it landed in time. |

Poll `GET /chats/{chatId}/messages/{messageId}/status` for the latest. For real-time, prefer webhooks (see `blooio-webhooks` skill) — `message.sent`, `message.delivered`, `message.failed`, `message.read`.

## Send a text

```http
POST /chats/%2B15551234567/messages
Idempotency-Key: 6f1e...-uuid
{ "text": "Hi from Blooio" }
```

`text` can also be an array — each entry becomes its own message and the response returns `message_ids[]` plus `count`.

## Send with an attachment

Attachments are referenced by **public HTTPS URL** (not bytes). Blooio fetches them server-side.

```json
{
  "text": "Receipt attached",
  "attachments": [
    { "url": "https://example.com/r.pdf", "name": "receipt.pdf" }
  ]
}
```

Up to 16 MB per asset. Don't send attachment-only as the **first** message to a brand-new contact — you'll get `attachment_first_message_not_allowed`. Send a text first.

## Multipart (single bubble vs URL-balloon batch)

Use `parts[]` when you want a unified iMessage bubble that mixes text and attachments:

```json
{
  "parts": [
    { "text": "Latest specs:" },
    { "url": "https://example.com/spec.pdf", "name": "spec.pdf" }
  ]
}
```

Add a `mention` to render an `@`-tag inside a part (group chats only):

```json
{ "parts": [{ "text": "Heads up", "mention": "+15551234567" }] }
```

If **any** part has a `link_preview`, you've opted into **URL-balloon batch mode**: every part must be text-only and contain a single http(s) URL, and each becomes its own rich-link iMessage. The response shape changes — `message_ids[]` + `count` instead of `message_id`.

## Link-preview override

By default Blooio auto-fetches OpenGraph metadata for URL messages. Override it:

```json
{
  "text": "https://blooio.com",
  "link_preview": {
    "title": "Blooio — iMessage at scale",
    "image_url": "https://blooio.com/og.png"
  }
}
```

Image must be HTTPS PNG/JPG/WebP/GIF, ≤16 MB. If the download fails, send still succeeds and falls back to auto-OG.

## iMessage send-with-effect

Set `effect` on a top-level `text` send (not supported with `parts`).

- **Bubble effects:** `slam`, `loud`, `gentle`, `invisible-ink`
- **Screen effects:** `echo`, `spotlight`, `balloons`, `confetti`, `love`, `lasers`, `fireworks`, `celebration`

```json
{ "text": "🚀 Shipped v2", "effect": "fireworks" }
```

iMessage-only — silently downgraded to a plain message on SMS/RCS.

## Native polls

```http
POST /chats/%2B15551234567/polls
{ "title": "Lunch?", "options": ["Pizza", "Sushi", "Tacos"] }
```

Read results back at `GET /chats/{chatId}/polls/{pollId}`. Set `webhook_type=poll` (or `all`) to get `poll.received`, `poll.created`, `poll.voted` events. The MCP doesn't expose polls — call REST.

## Reactions (tapbacks + emoji)

Tapbacks: `+love`, `-love`, `+like`, `-like`, `+dislike`, `-dislike`, `+laugh`, `-laugh`, `+emphasize`, `-emphasize`, `+question`, `-question`. Emoji: any emoji prefixed with `+` or `-` (e.g. `+😂`, `-🔥`). Emoji reactions need macOS 14 Sonoma+ on the linked Mac.

```http
POST /chats/%2B15551234567/messages/-1/reactions
{ "reaction": "+love" }
```

`messageId` accepts an explicit `msg_xxx` or a relative index (`-1` = last message in chat, `-2` = second-to-last…). Combine with `direction: "inbound"` to react only to received messages.

## Typing & read receipts

```http
POST /chats/%2B15551234567/typing      # start
DELETE /chats/%2B15551234567/typing    # stop
POST /chats/%2B15551234567/read        # mark all as read
```

Typing is **iMessage-only** — RCS doesn't carry composing state. The API returns `200` with a `warning` field on RCS chats; the indicator just isn't visible.

## Multi-recipient sends (ad-hoc groups)

Pass a comma-separated, URL-encoded `chatId`. An unnamed group is auto-created (or reused if the same participant set already exists):

```http
POST /chats/%2B15551111111%2C%2B15552222222/messages
{ "text": "Standup in 5 ☕️" }
```

The response includes `group_id`, `group_created`, and `participants[]`.

## Choosing `from_number`

If the API key has multiple numbers, pass `from_number` to pin the send. Twilio API keys auto-select the first assigned number when omitted; Blooio (iMessage) keys round-robin across the pool unless pinned.

## Cancelling a queued send

`DELETE /chats/{chatId}/messages/{messageId}` while status is still `queued` or `pending`. Past `sent`, you can't unring the bell.

## What can go wrong

- **403 `inbound_only_no_prior_inbound`** — you're on the Inbound plan; the recipient must message first. Ask the user to share a Blooio number with the contact, or upgrade to Dedicated.
- **403 (no `code`)** — recipient is an emergency number, or the number is integration-managed and you can't manually send from it.
- **429 `outbound_limit_reached`** — org-level new-contact cap from Settings.
- **429 `new_conversation_limit_reached`** — shared-plan daily cap. Existing chats keep working.
- **502** — temporary device error. Retry with backoff.
- **503** — no active device. Surface to user; their Mac is offline/locked/needs reboot.

Switch on `code`, not on the human `message` string.
