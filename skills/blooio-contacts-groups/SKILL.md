---
name: blooio-contacts-groups
description: >-
  Manage Blooio contacts (phone/email, capabilities, tags) and groups (named
  groups, linking to existing iMessage chats via chat_guid, members, group
  icons). Use when the user is building an address book, segmenting contacts
  with tags, creating or syncing group chats, checking iMessage/SMS/FaceTime
  capabilities, or wiring contact data into a CRM.
---

# Blooio contacts & groups

## Contacts

A contact is a phone number (E.164) or an email address registered in your org. Use them to attach a name, tags, and capability info to an identifier.

### CRUD

```http
GET    /contacts?q=jane&sort=name_asc&limit=50&offset=0
POST   /contacts                                 # { "identifier": "+15551234567", "name": "Jane" }
GET    /contacts/{contactId}
PATCH  /contacts/{contactId}                     # { "name": "Jane Doe" } — null to clear
DELETE /contacts/{contactId}                     # soft delete
```

`contactId` is the URL-encoded identifier (`%2B15551234567` or `user%40example.com`). Conflict on create returns `409` with the existing contact body.

### Capabilities

```http
GET /contacts/{contactId}/capabilities
```

Returns `{ imessage, sms, facetime, last_checked }`. Use this **before** sending the first message to a brand-new contact so you can pick the right protocol and avoid surprising fall-back-to-SMS behavior. Result is cached server-side; `last_checked` tells you how stale it is.

### Tags

Tags are free-form strings — use them as your CRM segments (`vip`, `lead`, `paying`).

```http
GET    /contacts/{contactId}/tags
POST   /contacts/{contactId}/tags         # { "tags": ["vip", "priority"] }
DELETE /contacts/{contactId}/tags/{tag}
```

`POST` is idempotent — re-adding a soft-deleted tag re-activates it. Filter chats/contacts by tag client-side after listing — there's no `?tag=` query param yet.

## Groups

A "group" in Blooio is a named container of contacts. It can be **linked to an existing iMessage group chat** via `chat_guid`, or **stand-alone** (a new iMessage chat is created on first send).

### Two creation modes

```http
POST /groups
```

**Mode 1 — link to an existing iMessage group:**

```json
{
  "name": "Sales Team",
  "chat_guid": "iMessage;+;chat123456789",
  "members": ["+15551234567", "+15559876543"]
}
```

The `members` list here is **record-keeping only** — those identifiers are not added to the linked iMessage chat (use Group Members add/remove for that, when those endpoints come out of "Coming Soon"). You get `chat_guid` from `BlueBubbles` API or from any inbound message webhook for that chat. Multiple groups can share participants if their `chat_guid`s differ.

**Mode 2 — new group:**

```json
{ "name": "Beta Testers", "members": ["+15551234567", "alice@example.com"] }
```

A new iMessage chat is created on first send. Note: iMessage only allows **one chat per unique participant set** when created via API. If a chat with the same participants exists, you get `409 Conflict`.

### Reading

```http
GET /groups?q=sales&sort=recent&limit=50
GET /groups/{groupId}
GET /groups/{groupId}/members
```

Group response includes `member_count`, `message_count`, `last_message_text`, `last_message_time`, `last_message_direction`, and `icon_url` (if set).

### Updating

```http
PATCH /groups/{groupId}                # { "name": "Sales Team — North" }
DELETE /groups/{groupId}               # soft delete; member rows go too
```

If the group has a `chat_guid`, renaming pushes through to the linked iMessage chat (display-name only — iMessage's one-chat-per-participant-set rule means it's the same thread). The `device_sync` block in the response tells you whether the linked-chat update succeeded.

### Group icons

```http
POST   /groups/{groupId}/icon          # multipart/form-data, field name `icon`
DELETE /groups/{groupId}/icon
```

Requires a `chat_guid`. The icon is stored in Blooio storage and synced to the linked iMessage chat before the call returns. JPEG/PNG/GIF/WebP/HEIC, ≤10 MB.

### Members (Coming Soon)

`POST /groups/{groupId}/members` and `DELETE /groups/{groupId}/members/{contactId}` exist in the spec but currently return `501` while Apple-side join/leave flows are stabilized. Track that in the dashboard changelog. In the meantime, members of a linked iMessage chat are managed on a real device (or via BlueBubbles).

### Sending to a group

Use the `groupId` as the `chatId`:

```http
POST /chats/grp_abc123def456/messages
{ "text": "Standup in 5" }
```

Or send to an ad-hoc group with comma-separated participants (see the `blooio-messaging` skill).

## Tag-based bulk sends

Pattern: list contacts, filter by tag client-side, send one message per chat with a unique `Idempotency-Key`. Throttle to avoid `outbound_limit_reached`.

```ts
const contacts = (await blooio("/contacts?limit=200")).contacts.filter(c => c.tags.includes("vip"));
for (const c of contacts) {
  await blooio(`/chats/${encodeURIComponent(c.identifier)}/messages`, {
    method: "POST",
    headers: { "Idempotency-Key": `weekly-${c.contact_id}-${weekStart}` },
    body: JSON.stringify({ text: "Weekly drop is live ✨" }),
  });
}
```

The idempotency key is **per-message**, so include the contact id and a stable per-blast key. That makes the whole loop safe to re-run without double-sending.
