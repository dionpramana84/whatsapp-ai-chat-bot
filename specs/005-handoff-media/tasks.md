# 005 — Handoff + media: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

## Group A — Schema and config

- [ ] HND-T01 Migration: `contacts.paused_at`, `contacts.last_agent_message_at`, `messages.sender`, `messages.media`, `messages.transcribed` + pending-media index
  - Refs: HND-FR-01, HND-FR-07
  - Files: src/db/schema.ts, drizzle/000X_*.sql
  - Done when: the migration applies on top of 004.

- [ ] HND-T02 Config: `HANDOFF_*`, `ADMIN_TOKEN` (≥ 32 chars), `MEDIA_*`, `STT_*`, `LLM_SUPPORTS_IMAGES`
  - Refs: HND-FR-01, HND-FR-03, HND-FR-04, HND-FR-07, HND-FR-09, HND-FR-10
  - Files: src/config.ts, .env.example, test/config.test.ts
  - Done when: the parity test passes. A short `ADMIN_TOKEN` fails startup.

## Group B — Handoff

- [ ] HND-T03 [P] Keyword matcher (whole word, case- and accent-insensitive): tests, then implementation
  - Refs: HND-FR-01
  - Files: src/handoff/keywords.ts, test/handoff/keywords.test.ts
  - Done when: HND-FR-01.AC2 passes, plus multi-language cases.

- [ ] HND-T04 Reply-worker handoff + pause tests (keyword path, tool path, paused silence)
  - Refs: HND-FR-01, HND-FR-02
  - Files: test/worker/handoff.test.ts
  - Depends: HND-T01, HND-T03
  - Done when: the tests exist and fail.

- [ ] HND-T05 Implement keyword pre-check, the `request_human` tool, the pause transaction and paused silence in the reply worker
  - Refs: HND-FR-01, HND-FR-02
  - Files: src/worker/reply.ts, src/tools/index.ts, src/handoff/pause.ts
  - Depends: HND-T04
  - Done when: HND-FR-01.AC1 and HND-FR-02.AC1 pass.

- [ ] HND-T06 `notify` queue + Slack/Discord-compatible sender with masked `wa_id`: tests, then implementation
  - Refs: HND-FR-03
  - Files: src/handoff/notify.ts, src/queue/queues.ts, test/handoff/notify.test.ts
  - Depends: HND-T05
  - Done when: HND-FR-03.AC1 passes.

## Group C — Admin API

- [ ] HND-T07 Admin auth middleware + route tests (401s, pagination cap, 409 on closed window, resume)
  - Refs: HND-FR-04
  - Files: test/http/admin.test.ts
  - Depends: HND-T01
  - Done when: the tests exist and fail.

- [ ] HND-T08 Implement the `/admin` sub-app (list, messages, agent send, resume) + rate limit
  - Refs: HND-FR-04, HND-NFR-02
  - Files: src/http/admin/*.ts, src/http/app.ts
  - Depends: HND-T07
  - Done when: HND-FR-04.AC1–AC3 pass.

- [ ] HND-T09 Maintenance schedule: auto-resume
  - Refs: HND-FR-05
  - Files: src/maintenance/auto-resume.ts, src/queue/queues.ts, test/maintenance/auto-resume.test.ts
  - Depends: HND-T05
  - Done when: HND-FR-05.AC1 passes (time is injected).

## Group D — Media

- [ ] HND-T10 [P] Media fixtures (image, audio, PDF, oversized, wrong magic bytes) + fake Graph media endpoints
  - Refs: HND-FR-07
  - Files: test/fixtures/media/*, test/helpers/fake-graph.ts
  - Done when: the fixtures are committed (small files only, < 200 KB total).

- [ ] HND-T11 Media job tests (happy path, size cap, MIME/magic mismatch, 404 expired, 5xx retry with meta re-fetch, sha256 mismatch)
  - Refs: HND-FR-07
  - Files: test/media/download.test.ts
  - Depends: HND-T01, HND-T10
  - Done when: the tests exist and fail.

- [ ] HND-T12 Implement the media job + webhook enqueue for media messages
  - Refs: HND-FR-07, HND-NFR-01
  - Files: src/media/download.ts, src/http/webhook/route.ts, src/queue/queues.ts
  - Depends: HND-T11
  - Done when: HND-FR-07.AC1–AC3 pass.

- [ ] HND-T13 Reply-worker deferral while media is pending: tests, then implementation
  - Refs: HND-FR-08
  - Files: src/worker/reply.ts, test/worker/media-deferral.test.ts
  - Depends: HND-T12
  - Done when: HND-FR-08.AC1 passes, including the 15-deferral cap.

- [ ] HND-T14 STT (`google` audio input; `whisper_local` optional) with a duration cap + failure fallback
  - Refs: HND-FR-09
  - Files: src/media/stt.ts, test/media/stt.test.ts
  - Depends: HND-T12
  - Done when: HND-FR-09.AC2 passes with a fake. AC1 is verified live in HND-T18.

- [ ] HND-T15 Model assembly for audio, image, location + fallbacks for no-vision, document, video, contacts, sticker, unsupported
  - Refs: HND-FR-10, HND-FR-11
  - Files: src/llm/build-messages.ts, src/worker/compose-*.ts, test/llm/build-messages-media.test.ts
  - Depends: HND-T13, HND-T14
  - Done when: HND-FR-10.AC1 and every HND-FR-11 per-type test pass.

- [ ] HND-T16 [P] Media retention cleanup (scheduled daily)
  - Refs: HND-FR-12
  - Files: src/maintenance/media-cleanup.ts, test/maintenance/media-cleanup.test.ts
  - Depends: HND-T12
  - Done when: HND-FR-12.AC1 passes.

## Group E — Inbox decision (blocked)

- [ ] HND-T17 Resolve HND-FR-06. Write a follow-up spec (`007-agent-inbox`) for Chatwoot integration or a custom TanStack Start inbox.
  - Refs: HND-FR-06
  - Done when: the decision is recorded in this spec with its rationale.

## Group F — Manual end-to-end

- [ ] HND-T18 Live test on the test number
  - Refs: US1–US5
  - Depends: Groups B–D
  - Done when (record in the PR): "talk to a human" produces the acknowledgement + a Slack/Discord notification. Further messages get silence. An agent send through the admin API arrives on the phone. Resume makes the bot reply again. A voice note is answered by its content. An image is described (vision model) or gets the fallback. A PDF gets the fallback.

## Exit criteria

- Groups A–D and F done, CI green. Group E decided.
