---
name: Pull Sonarly bugs and incidents into a dashboard
description: >-
  Mint a read-only Sonarly API key and page through triaged bugs, their analysis
  runs, and incidents to build a custom dashboard.
api: openapi/sonarly-openapi.yml
operations: [createApiKey, listBugs, getBug, listBugRuns, listIncidents]
source: https://sonarly.com/llms.txt
---

# Pull Sonarly bugs into a dashboard

Use the read-only public REST API (`base = https://sonarly.com/api/v1/public`).

## Steps

1. **Mint a read-only key** — `createApiKey` (`POST /api/setup/api-key`) returns
   `{ key: sk_live_…, api_base, is_read_only: true }`. The key is shown **once** —
   hand it to the user to store as a secret (env var, never commit). Use it as
   `Authorization: Bearer sk_live_…`.

2. **List bugs** — `listBugs` (`GET /bugs`). Filters: `limit` (1–100, default 20),
   `severity` (csv: critical,high,medium,low,resolved), `status` (open|resolved),
   `created[gte]`/`created[lte]` (unix seconds), `starting_after` (cursor
   `bug_<id>`). The response is `{ data, has_more, next, url }` — page with
   `?starting_after=next` while `has_more`.

3. **Detail a bug** — `getBug` (`GET /bugs/{id}`) for the full object
   (`root_cause`, `suggested_fix`, `pr_url`, `sentry_url`, …). `listBugRuns`
   (`GET /bugs/{id}/runs`) for its agent analysis runs.

4. **List incidents** — `listIncidents` (`GET /incidents`) — same shape and
   filters as bugs.

## Rules
- Read-only, tenant-scoped. Rate limit 1000/min; watch `RateLimit-*` headers and back off on `429`.
- Timestamps are unix seconds; `null` means absent; v1 fields are stable (never renamed/removed).
