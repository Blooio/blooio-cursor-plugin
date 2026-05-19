---
name: blooio-webhooks
description: >-
  Subscribe to Blooio webhook events, verify signatures, debug delivery, and
  replay events. Covers webhook_type (message/status/poll/all), payload
  schema, signing-secret rotation, delivery logs, and replay. Use when the
  user is wiring up a webhook receiver, debugging missing events, getting
  signature failures, or asking about which event types to subscribe to.
---

# Blooio webhooks

Webhooks are how you get inbound messages, delivery state changes, and poll events out of Blooio in real time. Up to 64 webhooks per organization.

## Subscribe

```http
POST /webhooks
{
  "webhook_url": "https://example.com/hooks/blooio",
  "webhook_type": "all",
  "valid_until": -1
}
```

`webhook_type`:
- `message` — inbound and outbound message events (`message.received`, `message.sent`, `message.delivered`, `message.failed`, `message.read`)
- `status` — only outbound status changes (sent/delivered/failed/read)
- `poll` — `poll.received`, `poll.created`, `poll.voted`
- `all` — everything

`valid_until`: epoch ms when the subscription auto-expires. Use `-1` for no expiration.

The response on `201` includes `signing_secret`. **Show it once, store it securely** — Blooio never returns it again. If you lose it, rotate via `POST /webhooks/{webhookId}/secret/rotate` (the old secret invalidates immediately).

If the URL is already subscribed, you get `200` (idempotent) without a new secret — that's normal, reuse the original.

## Receive: payload shape

Every event has at least:

```json
{
  "event": "message.delivered",
  "message_id": "msg_abc123",
  "external_id": "+15551234567",
  "internal_id": "+15559998888",
  "status": "delivered",
  "protocol": "imessage",
  "timestamp": 1716222000123,
  "is_group": false
}
```

For `is_group: true` you also get `group_id`, `group_name`, and `participants[]`. For `message.received` events you get `text`, `attachments[]`, and `sender`. For `message.failed` you get `error_code` + `error_message`.

The full schema is `WebhookEventPayload` in the [OpenAPI spec](https://backend.blooio.com/v2/api/openapi.json).

## Verify the signature

Blooio signs every delivery with the webhook's `signing_secret`. Compare the request signature header against an HMAC of the raw body **before parsing the JSON**:

```ts
import crypto from "node:crypto";

export function verifyBlooioSignature(rawBody: Buffer, header: string, secret: string) {
  const expected = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(expected, "utf8");
  const b = Buffer.from(header, "utf8");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

Fail closed: if verification fails, return `401` and don't process the body. Always read the **raw bytes** before any framework JSON parsing — Express needs `app.use(express.raw({ type: "application/json" }))` on the webhook route.

## Make your handler idempotent

Blooio retries on non-2xx and on timeout. Dedupe on `message_id` + `event` (or on the `event_id` exposed in the delivery log). A row in your DB keyed on `(message_id, event)` with `INSERT ... ON CONFLICT DO NOTHING` is enough.

Return a 2xx fast — Blooio doesn't read your response body for content. Long handler? Enqueue and ack within ~5 seconds.

## Debug: delivery logs

```http
GET /webhooks/{webhookId}/logs?limit=50&sort=desc
GET /webhooks/{webhookId}/logs?min_status=400&max_status=599
```

Each log entry contains `event_id`, `webhook_url`, the full `event_body` we tried to deliver, the `response_status` we got back, and the response JSON we parsed. Filter by status code to find failures fast.

## Replay an event

```http
POST /webhooks/{webhookId}/logs/{eventId}/replay
```

Re-sends the original payload (with a fresh `event_id`). The response tells you whether the replay got a 2xx, the status code, and how long it took. Useful to recover from outages on your side without losing data.

## Lifecycle: deprecate vs delete

`PATCH /webhooks/{webhookId}` with `deprecate: true` pauses delivery but keeps logs. `DELETE` removes the webhook permanently. Prefer deprecate during incident response so you can replay backlogged events afterwards.

## Tuning

- Public, HTTPS-only endpoints. `http://` URLs are rejected.
- `failure_count` on the webhook resource increments on non-2xx responses; track it from your monitoring.
- After repeated failures, Blooio backs off but does not auto-disable. Either fix or `deprecate: true`.

## Common pitfalls

- **JSON parsed before signature checked** — body bytes change after parse/serialize round-trip; HMAC won't match. Read raw bytes first.
- **Subscribing to `message`, expecting polls** — switch to `poll` or `all`.
- **Not URL-encoding the chatId** when calling back into the API from your webhook. The `external_id` field is unencoded; encode it before using it as a path segment.
- **Slow handlers** — webhook timeouts retry the whole event, doubling load. Acknowledge fast, do work async.
