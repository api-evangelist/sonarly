---
name: Onboard a Sonarly tenant from a coding agent
description: >-
  Set up Sonarly for a user from inside their repository — detect the stack,
  create the tenant, connect code + error tracking, wire observability, and
  finish — doing every step yourself except the OAuth browser clicks you hand
  off to the human.
api: openapi/sonarly-openapi.yml
operations: [startSetup, getSetupStatus, getConnectUrl, listRepos, selectRepos, createIntegration, configureProject, completeSetup]
source: https://sonarly.com/llms.txt
---

# Onboard a Sonarly tenant

Sonarly's onboarding is agent-native: you do every step except the OAuth browser
consent screens, which you hand to the human as a link. Never ask for a password;
never ask what you can detect from the repo.

## Steps

1. **Detect the stack locally** — read `package.json` / `pyproject.toml` / `go.mod`
   / `Gemfile` etc. for language, framework, package manager, git host
   (`git remote -v`), monorepo layout, and already-instrumented tools
   (`@sentry/*`, `dd-trace`, OpenTelemetry, etc.). Keep this stack profile.

2. **Start the session** — `startSetup` (`POST /api/setup/start`) with
   `{ source, stack }`. Returns `setup_id`, `verification_url`, `user_code`,
   `interval`, `expires_in`. Show the human the `verification_url` to sign in.

3. **Poll for authorization** — `getSetupStatus`
   (`GET /api/setup/{setup_id}/status`) respecting `interval`, until
   `state=authorized`. Capture the setup-scoped `token`; use it as
   `Authorization: Bearer <token>` for every call below. Restart on `expired`.

4. **Connect the code host** — `getConnectUrl?provider=github`, show the install
   URL, poll status until `connected.github` is true. Then `listRepos`
   (`GET /api/setup/repos`) and `selectRepos`
   (`POST /api/setup/repos/selected`) — pick the repos yourself (default the
   current `origin`).

5. **Connect error tracking** — `getConnectUrl?provider=sentry`, show URL, poll
   until `connected.sentry` is true.

6. **Wire observability (API-key backends)** — for each tool the stack profile
   suggests, `createIntegration` (`POST /api/setup/{id}/integrations`) with
   `{ backend_type, config }`. On success Sonarly auto-provisions the webhook.

7. **Configure the project** — `configureProject` (`POST /api/setup/project`)
   with `agent_instructions` written from the codebase (framework, package
   manager, dirs to avoid, test command, conventions).

8. **Finish** — `completeSetup` (`POST /api/setup/complete`); give the user the
   returned `dashboard_url`.

## Rules
- `[human]` = one OAuth click you hand off; `[agent]` = an API call you make with the token.
- The setup token is short-lived (~30 min) and scoped to this one tenant's setup endpoints.
- GitLab and non-Sentry error trackers return `400` today (wired incrementally) — surface, don't retry blindly.
