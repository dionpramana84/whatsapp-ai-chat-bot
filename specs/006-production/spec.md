# 006 — Production hardening + go-live

- **Status:** Draft (clarifications open)
- **Phase:** 5
- **Depends on:** 004, 005

## Goal

Make the bot safe to put in front of real customers: resilient to platform limits, observable, cost-tracked, load-tested, compliant with Meta's opt-in and template rules, deployed on always-on hosting, and switched from the test number and free tiers to production accounts through an explicit checklist.

This is the only spec where spending money is allowed (constitution VIII). Every paid item is listed in PRD-FR-10.

## User stories

- **US1:** As an operator, I find out about failures (Graph errors, LLM errors, stuck queues) before customers complain.
- **US2:** As an operator, I know what each conversation costs.
- **US3:** As the business, I can re-engage customers who opted in, with approved templates.
- **US4:** As the business, I can go live knowing every Meta and legal requirement is met.

## Functional requirements

### PRD-FR-01 — Graph API resilience
The WhatsApp client MUST classify errors by HTTP status and Graph error code, at minimum: throughput limit (`130429`), pair rate limit (`131056`), outside window (`131047`), undeliverable (`131026`), auth errors (`190`). Throughput and pair limits MUST back off (pair limit: wait at least 6 s before the next message to the same recipient). Auth errors MUST alert immediately and pause sending. **Verify every code against Meta's current error reference.**

- **AC1:** Given a `131056` response, when retried, then the next attempt waits at least 6 s.
- **AC2:** Given a `190` response, when received, then an alert fires and the reply queue stops pulling until an operator resumes it.

### PRD-FR-02 — Error tracking and alerts
Unhandled errors and failed jobs (after the last retry) MUST be reported to Sentry, with secrets and message bodies scrubbed. Alerts MUST fire for: auth errors, a job failure rate over 5% in 15 min, queue depth over 100 or oldest job older than 2 min, and LLM fallback rate over 10% in 15 min.

- **AC1:** Given a test exception containing a token-shaped string, when it is reported, then the Sentry event has no token. (unit, on the `beforeSend` scrubber)

### PRD-FR-03 — Metrics and cost
The app MUST record per-conversation cost: the LLM cost (tokens × the configured price table) and the Meta message cost (category × the configured rate card), visible through `GET /admin/metrics` (daily totals, top 20 conversations by cost) and in logs.

- **AC1:** Given 3 LLM turns with known token counts and a price table, when metrics are computed, then the cost matches the hand calculation to the cent.

### PRD-FR-04 — Opt-in and templates
`contacts.opted_in_at` and `contacts.opt_in_source` MUST exist. Business-initiated template sends MUST be refused for contacts without opt-in. The customer MUST be able to opt out (keywords `STOP`, `BERHENTI`, configurable), which sets `opted_out_at`, sends a confirmation, and blocks all further templates. Re-engagement templates MUST be defined in `templates/*.json`, submitted through the Meta dashboard, and referenced by name and language.

- **AC1:** Given a contact without opt-in, when a template send is attempted, then it's refused with `no_opt_in`.
- **AC2:** Given "STOP", when received, then `opted_out_at` is set and a confirmation is sent once.

### PRD-FR-05 — Load test
A k6 script MUST simulate signed Meta webhooks: 50 concurrent senders, bursts of 1–5 messages, 10 minutes, plus a 3× duplicate-delivery rate. Against a staging deployment with a mocked Graph API and a mocked LLM, it MUST show zero duplicate replies, zero lost messages, per-sender ordering kept, and webhook p95 under 300 ms.

- **AC1:** The load test report is committed to `loadtest/results/` and meets every threshold. (manual)

### PRD-FR-06 — Deployment
`[NEEDS CLARIFICATION: Production host and budget?]` Whatever the choice, the deployment MUST: run `web` and `worker` as separate always-on processes (the same image with different `ROLE`), run migrations as a release step before the new version receives traffic, use managed Postgres with pgvector, daily backups and a tested restore, provide secrets through the host's secret store, and terminate TLS at the platform.

- **AC1:** A restore from the latest backup into a scratch database succeeds and `npm run db:migrate` reports no pending migrations. (manual, quarterly)

### PRD-FR-07 — Container image
A multi-stage `Dockerfile` MUST produce a non-root image with production dependencies only, a `HEALTHCHECK` on `/healthz`, and `SIGTERM` handled as in FND-FR-08.

- **AC1:** Given `docker run` with `ROLE=web`, when `/healthz` is hit, then it returns 200 and the process runs as non-root.

### PRD-FR-08 — Data retention and deletion
Messages older than `MESSAGE_RETENTION_DAYS` (default 365) MUST be deleted by a scheduled job. An admin endpoint `DELETE /admin/contacts/:waId` MUST erase the contact, its messages, its media files and any derived data (within 24 h), for data-subject requests.

- **AC1:** Given a contact with messages and media, when the deletion runs, then no row or file referencing that `wa_id` remains.

### PRD-FR-09 — Secret rotation
The runbook MUST document rotating the system-user token, app secret, LLM keys and `ADMIN_TOKEN` without downtime (the app reads secrets at boot, so a rolling restart is enough).

- **AC1:** A rotation dry-run on staging is recorded. (manual)

### PRD-FR-10 — Go-live checklist
`docs/go-live.md` MUST be completed and signed off before switching. It contains at least:

1. Meta Business verification completed.
2. Production phone number registered (not active on the WhatsApp app, or migrated off it). Display name approved.
3. Payment method added to the WhatsApp Business Account. **Current rate card checked** (service-message pricing changed in 2026) and cost estimate approved.
4. Messaging limit tier understood. Plan for scaling up the tier.
5. Production LLM provider chosen (`[NEEDS CLARIFICATION]`), on a paid tier with no training on inputs, and the full eval suite passing at the 003/004 thresholds on it.
6. Production embedding model chosen. If it differs from the test one: migration + `kb:ingest --rebuild`.
7. `ALLOWED_WA_IDS` emptied (constitution IX no longer binds once no free tier is configured: confirm every provider is on a paid tier).
8. Privacy notice published and linked in the WhatsApp business profile. Retention values confirmed.
9. Re-engagement templates approved. Opt-in capture is live at every entry point.
10. Alerts tested by firing a synthetic error.
11. Load test passed (PRD-FR-05).
12. Rollback plan: re-point the webhook to the test app, or pause the reply queue.

- **AC1:** Every checklist item is ticked with the date and owner. (manual)

## Non-functional requirements

- **PRD-NFR-01:** Availability target 99.5% monthly for webhook acknowledgement. Meta retries cover short outages.
- **PRD-NFR-02:** Webhook p95 under 300 ms under load. p95 time to reply under `DEBOUNCE_SECONDS + 10 s`.

## Out of scope

- Multi-number / multi-tenant support.
- Payload CMS knowledge-base admin (its own spec when needed).

## Open questions

- `[NEEDS CLARIFICATION: Production host + budget]` (blocks PRD-FR-06)
- `[NEEDS CLARIFICATION: Production LLM provider]` (blocks checklist item 5)
- `[NEEDS CLARIFICATION: Applicable data-protection law (e.g. Indonesia's PDP Law, GDPR)?]` This sets the retention and deletion values.
