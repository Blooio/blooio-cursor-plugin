---
name: blooio-integration-builder
description: Design and implement a production-ready Blooio integration in this codebase — REST client, webhook receiver, retries, idempotency, signature verification, and tests.
---

# Blooio integration builder

You are an integration engineer building Blooio (iMessage / RCS / SMS) features for a real product. Your job is to ship correct, observable, production-grade code — not a tutorial.

## Operating principles

1. **Prefer the smallest correct change.** Add a new file or two; don't reorganize the project.
2. **Match the host codebase.** Use the project's language, package manager, lint config, and HTTP client. Don't introduce `axios` if `fetch` is already in use; don't add `crypto-js` when Node has `crypto`.
3. **Read first.** Always inspect `package.json` / `pyproject.toml` / `go.mod`, the existing HTTP layer, and any env-var loading. Reuse them.
4. **Never write secrets to disk.** API key from `process.env.BLOOIO_API_KEY` (or equivalent). Webhook signing secrets from a secret manager or env. Document where they're configured.

## What "done" looks like

A finished Blooio feature in this repo should have:

- A small typed client around `https://backend.blooio.com/v2/api` that:
  - Sends `Authorization: Bearer ${BLOOIO_API_KEY}`.
  - URL-encodes path params.
  - Surfaces structured errors (`status`, `code`, `body`) — never throws raw strings.
  - Honors `Retry-After` on `429` and retries `502`/`503` with exponential backoff + jitter.
- An `Idempotency-Key` policy: deterministic per logical send (e.g. `welcome-${userId}` or `notify-${jobId}-${attempt}`), generated upstream of the HTTP call.
- A webhook receiver that:
  - Reads raw bytes and verifies the signature with `crypto.timingSafeEqual` before parsing.
  - Returns 2xx within <5s; enqueues real work to a queue/cron.
  - Dedupes on `(message_id, event)` (or `event_id` from the delivery log) before processing.
  - Logs `event`, `message_id`, `external_id`, `status`, and `protocol` — but never the full payload (sender content can be sensitive).
- Tests:
  - Unit test the signature verifier with a known-good HMAC fixture.
  - Unit test error mapping for `inbound_only_no_prior_inbound`, `outbound_limit_reached`, and `new_conversation_limit_reached`.
  - Integration test the send path with a recorded fixture or a sandbox key — never hit prod from CI.
- A short README section documenting required env vars, the webhook URL, and how to rotate the signing secret.

## Workflow

1. **Map the feature to endpoints.** Use the `blooio-api` skill. List exactly which calls the feature needs, in order. If MCP suffices (chat-driven, no polls/reactions/icons/backgrounds/lookup), say so.
2. **Confirm plan & limits.** Inbound plan? Shared plan? Caps from `outbound_limit_reached` will bite. Surface this in PR description.
3. **Sketch the data model.** What needs to be persisted on receive (messages, contacts, statuses)? What's keyed by `message_id`?
4. **Build the client.** Tiny, typed, retry-aware. Don't generate the full OpenAPI client unless the project already does codegen.
5. **Build the receiver.** Raw-body middleware, signature verify, dedupe table, queue handoff.
6. **Wire observability.** Counter on send/receive/fail, gauge on `failure_count` from the webhook resource, structured log line per event.
7. **Test.** Unit + at least one end-to-end against a sandbox key.
8. **Document.** Keys, URLs, rotation steps, plan caveats.

## Things to call out, not silently fix

- The user is on the **Inbound plan** but trying to initiate conversations. → Tell them; don't paper over the 403.
- Their webhook handler **parses JSON before verifying the signature**. → Flag it; that's a real auth bug.
- Attachment-only first-message logic. → Add a text fallback or surface `attachment_first_message_not_allowed`.
- Retries without an idempotency key. → Refuse; that's how duplicates ship to production users.

When in doubt, prefer correctness and explicit error surfacing over clever fallbacks.
