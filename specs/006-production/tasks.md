# 006 — Production hardening + go-live: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

> Groups A–D can be built and verified at $0 (local/staging with mocks). Group E needs the hosting decision. Group F is go-live.

## Group A — Resilience

- [ ] PRD-T01 Verify Graph error codes against Meta's current reference. Write `src/whatsapp/errors.ts` policy table + tests.
  - Refs: PRD-FR-01
  - Files: src/whatsapp/errors.ts, test/whatsapp/errors.test.ts
  - Done when: each row of the design table has a test. The doc URL and check date are in a comment.

- [ ] PRD-T02 Pair-rate-limit delay and fatal-auth queue pause: tests, then implementation
  - Refs: PRD-FR-01
  - Files: src/whatsapp/client.ts, src/worker/reply.ts, src/http/admin/queue.ts, test/worker/rate-limit.test.ts
  - Depends: PRD-T01
  - Done when: PRD-FR-01.AC1–AC2 pass. `POST /admin/queue/resume` exists.

## Group B — Observability and cost

- [ ] PRD-T03 [P] Sentry integration with a `beforeSend` scrubber + test
  - Refs: PRD-FR-02
  - Files: src/observability/sentry.ts, test/observability/scrub.test.ts
  - Done when: PRD-FR-02.AC1 passes. Disabled when `SENTRY_DSN` is unset.

- [ ] PRD-T04 [P] Hashed `wa_id` in production logs (`LOG_SALT`)
  - Refs: PRD-FR-02
  - Files: src/logger.ts, test/logger.test.ts
  - Done when: in `NODE_ENV=production`, no raw `wa_id` appears in logs.

- [ ] PRD-T05 Alert evaluator (maintenance job) + `ALERT_WEBHOOK_URL`
  - Refs: PRD-FR-02
  - Files: src/maintenance/alerts.ts, test/maintenance/alerts.test.ts
  - Depends: PRD-T03
  - Done when: each threshold fires in a test with seeded pg-boss/messages data, and stays quiet below it.

- [ ] PRD-T06 Cost model: `config/pricing.ts`, `messages.cost` migration, compute on write, `GET /admin/metrics`
  - Refs: PRD-FR-03
  - Files: config/pricing.ts, src/db/schema.ts, drizzle/000X_*.sql, src/metrics/cost.ts, src/http/admin/metrics.ts, test/metrics/cost.test.ts
  - Done when: PRD-FR-03.AC1 passes.

## Group C — Compliance

- [ ] PRD-T07 Opt-in/opt-out: migration, keyword flows (opt-out first), template send guard: tests, then implementation
  - Refs: PRD-FR-04
  - Files: src/db/schema.ts, drizzle/000X_*.sql, src/compliance/opt.ts, src/whatsapp/client.ts, test/compliance/opt.test.ts
  - Done when: PRD-FR-04.AC1–AC2 pass.

- [ ] PRD-T08 [P] Template definitions + submission notes
  - Refs: PRD-FR-04
  - Files: templates/*.json, docs/templates.md
  - Done when: the templates follow Meta's category rules (utility vs marketing) and are reviewed. Submission happens in PRD-T16.

- [ ] PRD-T09 Retention job + contact erasure endpoint: tests, then implementation
  - Refs: PRD-FR-08
  - Files: src/maintenance/retention.ts, src/http/admin/contacts.ts, test/compliance/erasure.test.ts
  - Done when: PRD-FR-08.AC1 passes, including media files.

## Group D — Packaging and load

- [ ] PRD-T10 Multi-stage non-root Dockerfile + `.dockerignore`
  - Refs: PRD-FR-07
  - Files: Dockerfile, .dockerignore
  - Done when: PRD-FR-07.AC1 is verified (CI job builds the image and runs a container smoke test).

- [ ] PRD-T11 [P] Graph and LLM mock servers for staging/load
  - Refs: PRD-FR-05
  - Files: loadtest/mocks/graph.ts, loadtest/mocks/llm.ts
  - Done when: the mocks are configurable for latency and error rate. The app points at them through `GRAPH_BASE_URL` / the provider base URL config.

- [ ] PRD-T12 k6 load script + SQL assertions; run and commit the report
  - Refs: PRD-FR-05, PRD-NFR-02
  - Files: loadtest/webhook.js, loadtest/assert.sql, loadtest/results/*.md
  - Depends: PRD-T10, PRD-T11
  - Done when: PRD-FR-05.AC1 is met.

## Group E — Deployment (blocked on host decision)

- [ ] PRD-T13 Resolve PRD-FR-06 clarification. Record the decision and cost estimate in `spec.md`.
  - Refs: PRD-FR-06
- [ ] PRD-T14 Infra config (compose/fly.toml/railway.json), release migrate step, secrets, backups
  - Refs: PRD-FR-06
  - Depends: PRD-T10, PRD-T13
  - Done when: staging is deployed. PRD-FR-06.AC1 restore test is recorded.
- [ ] PRD-T15 [P] Rotation runbook + staging dry-run
  - Refs: PRD-FR-09
  - Files: docs/runbook.md
  - Depends: PRD-T14
  - Done when: PRD-FR-09.AC1 is recorded.

## Group F — Go-live

- [ ] PRD-T16 Write `docs/go-live.md` from PRD-FR-10 and work through it (business verification first, since it has the longest lead time: **start this during 003**)
  - Refs: PRD-FR-10
  - Done when: PRD-FR-10.AC1 is complete: every item dated and owned.
- [ ] PRD-T17 Production cutover: switch the webhook to the production app/number, empty the allowlist, watch alerts for 48 h
  - Refs: PRD-FR-10
  - Depends: PRD-T16
  - Done when: 48 h with no critical alerts. Spec status set to `Done`.
