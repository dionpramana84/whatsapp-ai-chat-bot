# 002 — Echo bot (proves the pipe)

- **Status:** Ready
- **Phase:** 1
- **Depends on:** 001

## Goal

Prove the full message pipeline end to end, with the hard parts done properly from the start: signature verification, idempotent persistence, per-sender serialization, retries and status tracking. Replies are a deterministic echo, so every failure here is a pipeline bug and not an LLM quirk.

## User stories

- **US1:** As a tester on an allowlisted number, when I send a text message, I get exactly one reply that echoes it.
- **US2:** As a tester, when I send three messages quickly, I get one reply covering all three, not three replies.
- **US3:** As an operator, when Meta retries a webhook or a job crashes halfway, no customer gets a duplicate or missing reply.
- **US4:** As an operator, I can see in the database whether each reply was sent, delivered, read or failed.

## Functional requirements

### ECH-FR-01 — Webhook verification handshake
`GET /webhook` MUST return `200` with the `hub.challenge` value as `text/plain` when `hub.mode=subscribe` and `hub.verify_token` equals `WHATSAPP_VERIFY_TOKEN` (constant-time comparison). Otherwise it MUST return `403`.

- **AC1:** Given the correct mode and token, when called, then the response is 200 and the body equals the challenge.
- **AC2:** Given a wrong token, when called, then the response is 403.
- **AC3:** Given a missing `hub.mode`, when called, then the response is 403.

### ECH-FR-02 — Signature verification
`POST /webhook` MUST compute HMAC-SHA256 of the **raw request body** with `WHATSAPP_APP_SECRET` and compare it in constant time with the `X-Hub-Signature-256` header (`sha256=<hex>`). A missing, malformed or mismatched signature MUST return `401` with no parsing and no database writes.

- **AC1:** Given a correctly signed body, when posted, then processing continues.
- **AC2:** Given a body signed with another secret, when posted, then the response is 401 and the `messages` table is unchanged.
- **AC3:** Given a missing header, when posted, then the response is 401.
- **AC4:** Given a header of a different length than expected, when posted, then the response is 401 and nothing throws.
- **AC5:** Given a correctly signed body that is re-serialized (whitespace changed), when posted with the original signature, then the response is 401. This proves the raw bytes are what gets verified.

### ECH-FR-03 — Payload validation and tolerant acknowledgement
Signed payloads MUST be validated against a schema that covers only the fields the bot uses, allowing unknown fields. A signed payload that doesn't match (an unknown event type) MUST be acknowledged with `200` and logged at `warn`, so Meta doesn't retry it forever.

- **AC1:** Given a signed payload for an unsubscribed field (for example `account_update`), when posted, then the response is 200 and nothing is persisted.

### ECH-FR-04 — Idempotent inbound persistence
For each inbound message, the webhook MUST upsert the contact (`wa_id`, profile name), insert the message with `ON CONFLICT (wamid) DO NOTHING`, and set `contacts.last_inbound_at` to the greater of its current value and the message's Meta timestamp.

- **AC1:** Given a new message, when posted, then one `messages` row exists with `direction='in'` and `processed_at IS NULL`.
- **AC2:** Given the same payload posted twice, when both complete, then exactly one row exists for that `wamid`.
- **AC3:** Given two messages delivered out of order, when both are stored, then `last_inbound_at` equals the later Meta timestamp.

### ECH-FR-05 — Enqueue reply work, recoverably
After persistence, the webhook MUST enqueue a `reply` job for every `wa_id` in the payload that has unprocessed inbound messages, with `singletonKey = wa_id`. It MUST do this **even when every insert was a conflict**, so that a retry after "insert succeeded, enqueue failed" still produces a reply.

- **AC1:** Given a new message, when posted, then one `reply` job exists for that `wa_id`.
- **AC2:** Given a message already stored but unprocessed and no job queued, when Meta retries the webhook, then a `reply` job is created.
- **AC3:** Given a message already processed, when Meta retries the webhook, then no new job is created.

### ECH-FR-06 — Acknowledge only after durable work
`POST /webhook` MUST return `200` only after persistence and enqueue succeed. If either fails, it MUST return `500` so Meta retries.

- **AC1:** Given the database throws during insert, when posted, then the response is 500.

### ECH-FR-07 — Status webhooks
Status events (`sent`, `delivered`, `read`, `failed`) MUST update the matching outbound message by `wamid`, and MUST never move a status backwards (`pending < sent < delivered < read`; `failed` overrides `pending` and `sent`). For `failed`, the error code and title MUST be stored. A status for an unknown `wamid` MUST be ignored.

- **AC1:** Given an outbound message with status `delivered`, when a late `sent` status arrives, then the status stays `delivered`.
- **AC2:** Given a `failed` status with error code 131047, when processed, then `status='failed'` and the `error` column holds the code.
- **AC3:** Given a status for an unknown `wamid`, when processed, then the response is 200 and nothing changes.

### ECH-FR-08 — One active reply job per sender
The `reply` queue MUST allow at most one active job and at most one queued job per `wa_id`. A message that arrives while a job is active MUST cause exactly one follow-up job.

- **AC1:** Given a job active for A, when two more messages arrive for A, then exactly one more job is queued for A.
- **AC2:** Given jobs for A and B, when the worker runs, then A and B are processed concurrently.

### ECH-FR-09 — Echo reply generation
The reply worker MUST load all unprocessed inbound messages for the `wa_id` in Meta timestamp order and produce **one** reply:
- text messages: `Echo: ` followed by the texts joined with newlines
- other types only: `Sorry, I can only read text messages for now.`
- reactions MUST NOT trigger a reply (they are marked processed)

- **AC1:** Given inbound texts "a", "b", when the job runs, then one outbound message `Echo: a\nb` is sent.
- **AC2:** Given one image message, when the job runs, then the fallback text is sent.
- **AC3:** Given only a reaction, when the job runs, then nothing is sent and the reaction is marked processed.

### ECH-FR-10 — Crash-safe send
The worker MUST, in one transaction, insert the outbound message with `status='pending'` and mark the inbound messages it consumed as processed, **then** call the Graph API. On retry, it MUST send any existing `pending` outbound message for that `wa_id` instead of generating a new one.

- **AC1:** Given the Graph call fails after the transaction committed, when the job retries, then the same outbound text is sent and no second outbound row is created.
- **AC2:** Given a successful send, when it completes, then the outbound row has the returned `wamid` and `status='sent'`.

### ECH-FR-11 — Graph API error handling
The WhatsApp client MUST treat HTTP 429, 5xx and network errors as retryable (throw, so pg-boss retries with backoff), and other 4xx as permanent (mark the outbound `failed` with the error and do not retry). Requests MUST time out after 10 s.

- **AC1:** Given the Graph API returns 500 then 200, when the job runs with retries, then the message is sent once.
- **AC2:** Given the Graph API returns 400 with error code 131047, when the job runs, then the outbound is `failed` and the job completes without retrying.
- **AC3:** Given the Graph API doesn't answer within 10 s, when the job runs, then it fails as retryable.

### ECH-FR-12 — Sender allowlist
When `ALLOWED_WA_IDS` is non-empty, messages from other `wa_id`s MUST be acknowledged with 200 and MUST NOT be stored or answered. An empty allowlist allows everyone, and the app MUST log a `warn` at startup when the allowlist is empty and `NODE_ENV !== 'production'`.

- **AC1:** Given `ALLOWED_WA_IDS=111`, when a message from 222 arrives, then the response is 200 and no row or job is created.

### ECH-FR-13 — Read receipts
The worker SHOULD mark the latest consumed inbound message as read through the Graph API before replying. A failure here MUST NOT fail the job.

- **AC1:** Given the mark-read call fails, when the job runs, then the reply is still sent.

### ECH-FR-14 — Privacy-aware logging
Logs MUST include `wamid` and `wa_id` for correlation. Message bodies MUST NOT be logged at `info` or above.

- **AC1:** Given an inbound text "secret", when processed at `LOG_LEVEL=info`, then no log line contains "secret".

## Non-functional requirements

- **ECH-NFR-01:** p95 `POST /webhook` latency under 300 ms locally (it does one transaction and one enqueue, no outbound HTTP).
- **ECH-NFR-02:** Replaying 100 identical signed webhooks concurrently results in exactly one reply.

## Out of scope

- LLM replies, debounce tuning, the 24-hour window (003).
- Media download (005).

## Open questions

None blocking.
