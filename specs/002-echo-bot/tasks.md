# 002 — Echo bot: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

## Group A — Pure building blocks

- [ ] ECH-T01 [P] Add webhook fixtures (text, image, reaction, multi-message, status sent/delivered/read/failed, account_update) + `signFixture` helper
  - Refs: ECH-FR-02, ECH-FR-03, ECH-FR-07
  - Files: test/fixtures/webhooks/*.json, test/helpers/sign.ts
  - Done when: fixtures match the payload shapes in Meta's webhook docs, with the source URL noted in a README in the folder.

- [ ] ECH-T02 [P] Signature verification: tests, then implementation
  - Refs: ECH-FR-02
  - Files: test/http/webhook/signature.test.ts, src/http/webhook/signature.ts
  - Done when: unit tests cover a valid signature, wrong secret, missing prefix, wrong length, non-hex, and a re-serialized body. All pass.

- [ ] ECH-T03 [P] Payload schema + `extractEvents`: tests, then implementation
  - Refs: ECH-FR-03
  - Files: test/http/webhook/payload.test.ts, src/http/webhook/payload.ts
  - Done when: every fixture parses into `{contacts, messages, statuses}`. `account_update` returns empty events. Types are inferred from zod.

- [ ] ECH-T04 [P] `composeEcho`: tests, then implementation
  - Refs: ECH-FR-09
  - Files: test/worker/compose-echo.test.ts, src/worker/compose-echo.ts
  - Done when: ECH-FR-09.AC1–AC3 pass as unit tests (pure function).

## Group B — WhatsApp client

- [ ] ECH-T05 WhatsApp client tests with a fake fetch (success, 500, 429, 400/131047, 400/130429, timeout, markRead)
  - Refs: ECH-FR-11, ECH-FR-13
  - Files: test/whatsapp/client.test.ts
  - Done when: the tests exist and fail.

- [ ] ECH-T06 Implement `createWhatsAppClient` with `RetryableError` / `PermanentError`
  - Refs: ECH-FR-11, ECH-FR-13
  - Files: src/whatsapp/client.ts
  - Depends: ECH-T05
  - Done when: ECH-T05 passes. The token never appears in thrown error messages.

## Group C — Ingest

- [ ] ECH-T07 Ingest integration tests (new message, duplicate, out-of-order timestamps, allowlist, status monotonicity, unknown status wamid)
  - Refs: ECH-FR-04, ECH-FR-07, ECH-FR-12
  - Files: test/http/webhook/ingest.test.ts
  - Depends: ECH-T03
  - Done when: the tests exist and fail.

- [ ] ECH-T08 Implement `persistInbound` + status update in one transaction; return wa_ids with unprocessed inbound
  - Refs: ECH-FR-04, ECH-FR-07, ECH-FR-12
  - Files: src/http/webhook/ingest.ts, src/db/messages.ts
  - Depends: ECH-T07
  - Done when: ECH-FR-04.AC1–AC3, ECH-FR-07.AC1–AC3 and ECH-FR-12.AC1 pass.

## Group D — Route

- [ ] ECH-T09 Webhook route integration tests (GET handshake, 401 cases, unknown event → 200, enqueue-on-conflict recovery, DB failure → 500)
  - Refs: ECH-FR-01, ECH-FR-02, ECH-FR-03, ECH-FR-05, ECH-FR-06
  - Files: test/http/webhook/route.test.ts
  - Depends: ECH-T02, ECH-T08
  - Done when: the tests exist and fail.

- [ ] ECH-T10 Implement `GET /webhook` and `POST /webhook` (raw body → verify → parse → ingest → enqueue → 200)
  - Refs: ECH-FR-01, ECH-FR-02, ECH-FR-03, ECH-FR-05, ECH-FR-06
  - Files: src/http/webhook/route.ts, src/http/app.ts, src/queue/queues.ts
  - Depends: ECH-T09
  - Done when: every AC for ECH-FR-01, 02, 03, 05 and 06 passes. Add `DEBOUNCE_SECONDS` (default 0) to config and `.env.example`.

## Group E — Worker

- [ ] ECH-T11 Reply worker integration tests (echo merge, pending replay after Graph failure, permanent failure, mark-read failure tolerated, concurrent workers don't double-consume)
  - Refs: ECH-FR-08, ECH-FR-09, ECH-FR-10, ECH-FR-11, ECH-FR-13
  - Files: test/worker/reply.test.ts
  - Depends: ECH-T04, ECH-T06, ECH-T08
  - Done when: the tests exist and fail.

- [ ] ECH-T12 Implement `handleReplyJob` and register the `reply` queue (stately policy, retries with backoff)
  - Refs: ECH-FR-08, ECH-FR-09, ECH-FR-10, ECH-FR-11, ECH-FR-13
  - Files: src/worker/reply.ts, src/queue/queues.ts, src/main.ts
  - Depends: ECH-T11
  - Done when: ECH-T11 passes. The pg-boss options are checked against the pinned version's docs, with a link in a code comment.

- [ ] ECH-T13 Per-sender serialization test (job active for A + 2 messages → exactly 1 queued; A and B run concurrently)
  - Refs: ECH-FR-08
  - Files: test/queue/reply-policy.test.ts
  - Depends: ECH-T12
  - Done when: ECH-FR-08.AC1–AC2 pass.

## Group F — Cross-cutting

- [ ] ECH-T14 [P] Log privacy test + logger fields (`wa_id`, `wamid` child loggers; bodies only at debug)
  - Refs: ECH-FR-14
  - Files: test/logging-privacy.test.ts, src/http/webhook/route.ts, src/worker/reply.ts
  - Depends: ECH-T12
  - Done when: ECH-FR-14.AC1 passes.

- [ ] ECH-T15 [P] Replay-storm test: 100 concurrent identical webhooks → 1 reply
  - Refs: ECH-NFR-02
  - Files: test/e2e/replay-storm.test.ts
  - Depends: ECH-T12
  - Done when: the test passes three times in a row (to catch flakiness).

- [ ] ECH-T16 Empty-allowlist startup warning
  - Refs: ECH-FR-12
  - Files: src/main.ts, test/main.test.ts
  - Depends: ECH-T10
  - Done when: a warning is logged when the allowlist is empty outside production.

## Group G — Manual end-to-end

- [ ] ECH-T17 Live test on the Meta test number over the ngrok tunnel
  - Refs: US1–US4
  - Depends: all above
  - Done when (record in the PR):
    1. Register the webhook URL and verify token in Meta. The handshake succeeds.
    2. Send "hello" from an allowlisted phone and get "Echo: hello" once.
    3. Send 3 messages within 1 s. With `DEBOUNCE_SECONDS=3`, get one merged reply.
    4. Send an image and get the fallback text.
    5. `messages` shows outbound statuses moving to `read`.
    6. Send from a non-allowlisted number (if available) and get no reply.

## Exit criteria

- Every box ticked. Every AC passing in CI. ECH-T17 recorded.
- Spec status set to `Done` in `specs/README.md`.
