---
name: blooio-api
description: >-
  Use the Blooio v2 REST API (https://backend.blooio.com/v2/api) for iMessage,
  RCS, and SMS automation. Covers auth, base URL, identifier formats, error
  codes, idempotency, plan limits, and the surface area (account, numbers,
  contacts, groups, chats, messages, reactions, polls, webhooks, location,
  facetime, phone-number lookup). Use when the user is writing code against
  Blooio, debugging API responses, choosing endpoints, or interpreting
  structured error codes.
---

# Blooio API (v2)

Production REST API for iMessage automation. Send messages, manage contacts/groups, subscribe to webhooks. The MCP server at `mcp.blooio.com/v4` is a thin wrapper over this same surface — when you need a feature the MCP doesn't expose (polls, reactions, group icons, chat backgrounds, FindMy location, FaceTime, phone-number lookup), call REST directly.

## Base URL & auth

- **Base URL:** `https://backend.blooio.com/v2/api`
- **Auth:** `Authorization: Bearer <BLOOIO_API_KEY>` on every request.
- **Spec:** `https://backend.blooio.com/v2/api/openapi.json` — keep this open while building. It's the source of truth for every field.

## Identifier formats

Get these wrong and you'll waste a lot of time on 400s:

- **Phone numbers** are always **E.164** with a leading `+`: `+15551234567`. Never spaces, dashes, or parentheses.
- **Path params containing `+` or `@` must be URL-encoded.** `+15551234567` → `%2B15551234567`. `user@example.com` → `user%40example.com`. The `chatId` path param is the most common offender.
- **Group IDs** look like `grp_abc123def456` (regex `^grp_[a-zA-Z0-9]+$`).
- **Webhook IDs** look like `wh_abc123def456`.
- **Multi-recipient chats** use a comma-separated, URL-encoded list as the `chatId`: `+15551234567,+15559876543` → `%2B15551234567%2C%2B15559876543`. Posting to that creates (or reuses) an unnamed group.

## High-level surface

| Tag | What it does | Key endpoints |
| --- | --- | --- |
| Account | Identity & device status | `GET /me`, `GET /me/numbers` |
| Contacts | Address book | `GET/POST /contacts`, `GET /contacts/{id}/capabilities`, `*/tags` |
| Groups | Named groups (optionally linked to an iMessage chat via `chat_guid`) | `POST /groups`, `*/icon`, `*/members` |
| Chats | Conversations (1:1 or group) | `GET /chats`, `GET /chats/{chatId}/messages` |
| Messages | Send & track | `POST /chats/{chatId}/messages`, `GET /chats/{chatId}/messages/{messageId}/status` |
| Reactions | Tapbacks + emoji | `POST /chats/{chatId}/messages/{messageId}/reactions` |
| Polls | Native iMessage polls | `POST /chats/{chatId}/polls`, `GET /chats/{chatId}/polls/{pollId}` |
| Typing / Read | UX signals | `POST/DELETE /chats/{chatId}/typing`, `POST /chats/{chatId}/read` |
| Chat Background | Conversation wallpaper | `GET/PUT/DELETE /chats/{chatId}/background` |
| Webhooks | Subscribe to events | `POST /webhooks`, `*/secret/rotate`, `*/logs`, `*/logs/{eventId}/replay` |
| Location | Cached FindMy positions | `GET /location/contacts`, `POST /location/contacts/refresh` |
| FaceTime | Initiate calls (Coming Soon) | `POST /facetime/calls` |
| Phone Numbers | Lookup, geocoding (Enterprise) | `GET/POST /phone-numbers/lookup`, `POST /phone-numbers/batch` |

## Idempotency

Pass `Idempotency-Key: <uuid>` on `POST /chats/{chatId}/messages` to make the request safe to retry. If the same key is replayed, the server returns the original message_id and a `200` instead of `202`. Always set this in production — networks fail and agents retry.

## Error codes (the only ones you should switch on)

The API returns a JSON `Error` body with `error`, `message`, `status`, and **`code`**. Switch on `code`, not `message` (the human-readable string can change):

- `inbound_only_no_prior_inbound` (403) — sender is on the Inbound plan and the recipient hasn't messaged you first. Body includes `allocation_id` and `external_id`. The number is reply-only by design; tell the user to upgrade or have the contact text first.
- `outbound_limit_reached` (429) — org-level new-contact cap. Body includes `limit`, `current`, `mode`, and on per-number mode `allocation_id` + `sender_number`.
- `new_conversation_limit_reached` (429) — shared plan's daily cap. Body includes `plan_id`, `cap`, `current`. Existing conversations keep working.
- `attachment_first_message_not_allowed` — first message to a brand-new contact can't be an attachment-only send. Send a text first.

Other status families:
- `400` — validation error. Read `message`. Most common cause: not URL-encoding the `chatId`.
- `401` — missing/invalid bearer token.
- `403` — generic forbidden (emergency number, integration-managed number you can't manually send from).
- `404` — endpoint or resource doesn't exist (e.g. unknown `messageId`).
- `429` (no `code`) — generic rate limit. Honor `Retry-After` if present.
- `502` — temporary device communication error. Safe to retry with backoff.
- `503` — no active device available. Surface to the user; usually their Mac needs attention.

## Pagination

Most list endpoints accept `limit` (1–200, default 50) and `offset`. The response contains a `pagination` object with `total`, `limit`, `offset`. Drive paging from `total` — don't just keep paging until empty (it's slower and races inserts).

## Typical implementation patterns

### Send a single text message

```http
POST /chats/%2B15551234567/messages
Authorization: Bearer ${BLOOIO_API_KEY}
Idempotency-Key: 7b3f...-uuid
Content-Type: application/json

{ "text": "Hello from Cursor" }
```

Response `202` with `{ "message_id": "msg_...", "status": "queued" }`. Poll `GET /chats/{chatId}/messages/{messageId}/status` until `status` is `delivered` (or `failed`).

### Multi-part with mention + attachment

```json
{
  "parts": [
    { "text": "@John see attached", "mention": "+15551234567" },
    { "url": "https://example.com/file.pdf", "name": "spec.pdf" }
  ]
}
```

### URL with link-preview override

```json
{
  "text": "https://news.ycombinator.com",
  "link_preview": {
    "title": "Hacker News",
    "image_url": "https://news.ycombinator.com/og.png"
  }
}
```

### Send-with-effect (iMessage only)

```json
{ "text": "🎉 Shipped!", "effect": "confetti" }
```

Falls back to a plain message on SMS/RCS.

## When to use REST vs MCP

- **MCP** — driving Blooio from chat / an agent. Send/receive, check delivery, manage webhooks.
- **REST** — production application code, anything that needs polls, reactions, group icons, contact capabilities, chat backgrounds, FindMy location, FaceTime, or phone-number lookup. The MCP doesn't surface those.

## Helper snippet (TypeScript)

```ts
const BLOOIO = "https://backend.blooio.com/v2/api";
async function blooio(path: string, init: RequestInit = {}) {
  const res = await fetch(`${BLOOIO}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${process.env.BLOOIO_API_KEY!}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });
  const body = await res.json().catch(() => ({}));
  if (!res.ok) {
    const err = new Error(body?.message ?? res.statusText) as Error & { code?: string; status: number; body: unknown };
    err.status = res.status;
    err.code = body?.code;
    err.body = body;
    throw err;
  }
  return body;
}

export const sendText = (to: string, text: string, idemKey: string) =>
  blooio(`/chats/${encodeURIComponent(to)}/messages`, {
    method: "POST",
    headers: { "Idempotency-Key": idemKey },
    body: JSON.stringify({ text }),
  });
```

Always `encodeURIComponent` the chatId. Always pass an idempotency key on send.
