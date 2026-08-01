---
name: Register and verify Sonarly webhooks
description: >-
  Register an outbound webhook receiver and verify Sonarly's signed event
  envelopes correctly (HMAC-SHA256, replay window, dedupe).
api: openapi/sonarly-openapi.yml
operations: [createWebhookEndpoint]
source: https://sonarly.com/llms.txt
---

# Register and verify Sonarly webhooks

## Steps

1. **Register the receiver** — `createWebhookEndpoint`
   (`POST /api/setup/webhook-endpoint`) with `{ url, events: ["*"] }` (or a
   subset). Returns `{ id, secret: whsec_…, event_types }`. The URL is
   SSRF-checked; `secret` is shown **once** — hand it to the user to set on their
   receiver.

2. **Verify every delivery (security-critical) — do ALL of these:**
   1. Capture the **raw body before JSON parsing** (Express `express.raw`;
      Next.js `await req.text()`; Workers `await req.text()`).
   2. Recompute `HMAC-SHA256(secret, "{t}.{rawBody}")` and constant-time compare
      to `v1` from the `Sonarly-Signature: t=<unix>,v1=<hex>` header.
   3. Reject if `|now - t| > 300s` (replay protection).
   4. Dedupe on `Sonarly-Event-Id` (stable across the 8-attempt retry schedule).
   Return `2xx` within 10s or Sonarly retries; persistent failures disable the
   endpoint after 3 days.

## Events
`bug.created, bug.analyzed, bug.deduped, bug.reappeared, bug.resolved,
bug.pr_created, bug.pr_merged, incident.created, incident.analyzed,
incident.resolved` (or subscribe `["*"]`).

## Envelope
`{ id: evt_…, type, api_version, created (unix), tenant_id, data: { object, previous_attributes } }`.
