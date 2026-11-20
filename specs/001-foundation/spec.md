# 001 — Foundation

- **Status:** Ready
- **Phase:** 0 (Setup)
- **Depends on:** —

## Goal

A developer can clone the repo, run the full stack locally with zero spend, and receive Meta webhooks on a stable public HTTPS URL, with the quality gates (types, lint, tests) enforced from the first commit.

## User stories

- **US1:** As a developer, I can start Postgres and the app with a couple of commands, so I can work without any cloud account except Meta's.
- **US2:** As a developer, I get a clear error at startup when configuration is missing or invalid, so I don't debug a half-configured app.
- **US3:** As a developer, I have a written runbook for creating the Meta test app, test number and system-user token, so setup is repeatable.
- **US4:** As a developer, Meta can reach my local webhook at a URL that doesn't change between restarts.

## Functional requirements

### FND-FR-01 — Toolchain
The repo MUST use TypeScript in strict mode on Node 24 LTS, with ESM modules, and expose `typecheck`, `lint`, `format:check` and `test` npm scripts.

- **AC1:** Given a fresh clone, when `npm ci && npm run typecheck && npm run lint && npm test` runs, then every command exits 0.
- **AC2:** Given a file that uses `any` explicitly, when lint runs, then it fails.

### FND-FR-02 — Continuous integration
A CI workflow MUST run typecheck, lint, format check, migrations and tests (against a real Postgres + pgvector service) on every push and pull request.

- **AC1:** Given a pull request, when CI runs, then all of these steps run and a failing test fails the workflow. (manual: open a PR with a deliberately failing test once)

### FND-FR-03 — Local database
Docker Compose MUST provide Postgres with the pgvector extension available, a healthcheck, and a persistent named volume.

- **AC1:** Given `docker compose up -d db`, when the healthcheck passes, then `CREATE EXTENSION IF NOT EXISTS vector` succeeds.

### FND-FR-04 — Validated configuration
The app MUST load configuration from environment variables, validated by a schema at startup. Invalid or missing required variables MUST stop the process with a non-zero exit code and a message naming each bad variable **without printing its value**.

- **AC1:** Given `WHATSAPP_APP_SECRET` is unset, when config loads, then it throws an error whose message contains `WHATSAPP_APP_SECRET`.
- **AC2:** Given `WHATSAPP_ACCESS_TOKEN=abc123` and an invalid `PORT`, when config loads, then the error message does not contain `abc123`.
- **AC3:** Given all variables are valid, when config loads, then a typed, frozen config object is returned.

### FND-FR-05 — Environment template
The repo MUST include `.env.example` listing every variable with a comment, and `.env*` (except `.env.example`) MUST be git-ignored.

- **AC1:** Given the config schema, when a test compares its keys with the keys in `.env.example`, then they match.

### FND-FR-06 — Schema migrations
The repo MUST manage the database schema with versioned migrations. `npm run db:migrate` MUST be idempotent and MUST enable the `vector` extension.

- **AC1:** Given an empty database, when `db:migrate` runs twice, then both runs succeed and the `contacts` and `messages` tables exist.

### FND-FR-07 — Health endpoint
The app MUST expose `GET /healthz`, returning `200 {"status":"ok"}` when the database answers a ping within 1 s, and `503` otherwise.

- **AC1:** Given the database is reachable, when `GET /healthz` is called, then the response is 200.
- **AC2:** Given the database client fails, when `GET /healthz` is called, then the response is 503 and the body has no stack trace.

### FND-FR-08 — Process roles
A single entry point MUST start the HTTP server, the queue worker, or both, depending on `ROLE` (`web` | `worker` | `all`, default `all`). On `SIGTERM`/`SIGINT` it MUST stop accepting new work, let in-flight work finish (up to 10 s), close the database pool and exit 0.

- **AC1:** Given `ROLE=web`, when the app starts, then no queue worker is registered.
- **AC2:** Given a running app, when it receives `SIGTERM`, then it exits with code 0 within 10 s.

### FND-FR-09 — Structured logging with redaction
The app MUST log JSON lines with a level from `LOG_LEVEL`, and MUST redact the access token, app secret, API keys and `authorization` headers wherever they appear in a logged object.

- **AC1:** Given a log call with `{ headers: { authorization: "Bearer x" } }`, when it is written, then the output contains no `Bearer x`.

### FND-FR-10 — Meta setup runbook
`docs/meta-setup.md` MUST describe, step by step: creating the Meta app and WhatsApp product, using the test number, adding and verifying up to 5 recipient numbers, creating a system user and a permanent token with `whatsapp_business_messaging` and `whatsapp_business_management`, finding the phone number ID and app secret, and subscribing the webhook to the `messages` field.

- **AC1:** Following the runbook from a fresh Meta developer account results in a working test number and token. (manual)

### FND-FR-11 — Stable tunnel
`docs/local-dev.md` MUST describe exposing the local server over HTTPS with an ngrok free static domain, and cloudflared as the fallback with its changing-URL caveat.

- **AC1:** With the tunnel running, `GET https://<static-domain>/healthz` returns 200. (manual)

## Non-functional requirements

- **FND-NFR-01:** No paid service is needed for anything in this spec (constitution VIII).
- **FND-NFR-02:** From a fresh clone to a green `npm test` takes under 10 minutes on a typical laptop.

## Out of scope

- Webhook handling (002).
- Production hosting and secret managers (006).

## Open questions

None blocking.
