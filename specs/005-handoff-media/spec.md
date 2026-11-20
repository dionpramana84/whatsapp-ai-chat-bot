# 005 — Human handoff + media

- **Status:** Draft (clarifications open)
- **Phase:** 4
- **Depends on:** 003 (004 recommended)

## Goal

Let a human take over a conversation without the bot talking over them, let agents reply, and handle voice notes and images instead of ignoring them. Meta's media URLs expire, so media is downloaded as soon as it arrives.

## User stories

- **US1:** As a customer, I can ask for a human and the bot stops replying while a person handles my conversation.
- **US2:** As an agent, I'm notified of a handoff and can reply to the customer.
- **US3:** As an agent, I can hand the conversation back to the bot.
- **US4:** As a customer, I can send a voice note and get an answer to what I said.
- **US5:** As a customer, I can send a photo (of a product or a problem) and get a relevant reply or a clear fallback.

## Functional requirements

### HND-FR-01 — Handoff trigger
The bot MUST hand off when (a) the model calls the `request_human` tool, or (b) the customer's message matches a configured keyword list (`HANDOFF_KEYWORDS`, default `human, agent, person, cs, manusia, admin`) as a whole word. Handoff MUST set `contacts.bot_paused = true` and `paused_at = now()`, send one acknowledgement (`HANDOFF_ACK_MESSAGE`), and notify agents.

- **AC1:** Given the message "can I talk to a human", when processed, then `bot_paused` is true, one acknowledgement is sent, and one notification is emitted.
- **AC2:** Given the message "humanity", when processed, then no handoff happens (whole-word match).

### HND-FR-02 — Silence while paused
While `bot_paused` is true, inbound messages MUST still be stored (and media downloaded) but MUST NOT produce bot replies. The reply job marks them processed without sending.

- **AC1:** Given a paused contact, when 3 messages arrive, then 3 are stored, 0 replies are sent, and all are processed.

### HND-FR-03 — Agent notification
Handoff notifications MUST go to `HANDOFF_NOTIFY_URL` (a Slack- or Discord-compatible incoming webhook, which is free) with the contact name, `wa_id` (masked to the last 4 digits) and a link to the admin conversation view. A failed notification MUST be retried by the queue and MUST NOT block the acknowledgement.

- **AC1:** Given the notify URL returns 500 then 200, when handoff happens, then the notification is delivered once and the acknowledgement was already sent.

### HND-FR-04 — Admin API (test-profile agent inbox)
The app MUST expose, under `/admin`, protected by a bearer `ADMIN_TOKEN` (constant-time compare, ≥ 32 chars):
- `GET /admin/conversations?paused=true`: paginated list (max 50 per page)
- `GET /admin/conversations/:waId/messages`: last 100 messages
- `POST /admin/conversations/:waId/messages` `{ text }`: send as agent (goes through the 24-hour guard, stored with `sender='agent'`)
- `POST /admin/conversations/:waId/resume`: `bot_paused = false`

- **AC1:** Given no or an incorrect token, when any `/admin` route is called, then the response is 401.
- **AC2:** Given a closed 24-hour window, when an agent sends, then the response is 409 with `window_closed` and nothing is sent.
- **AC3:** Given resume is called, when the next customer message arrives, then the bot replies again.

### HND-FR-05 — Auto-resume
A paused conversation MUST auto-resume after `HANDOFF_AUTO_RESUME_HOURS` (default 12) with no agent message. The bot MUST NOT reply retroactively to messages received while paused.

- **AC1:** Given a pause with no agent message for 12 h, when the scheduled check runs, then `bot_paused` is false, and the earlier messages stay processed with no reply.

### HND-FR-06 — Agent inbox UI
`[NEEDS CLARIFICATION: Chatwoot (self-hosted; needs Rails, Redis and Sidekiq, so heavier on a free VM) or a custom TanStack Start app on the admin API?]` Until it's decided, the admin API plus the notification is the inbox for testing.

### HND-FR-07 — Immediate media download
For inbound `audio`, `image`, `document` and `video` messages, the webhook-triggered pipeline MUST fetch the media metadata (`GET /{media-id}`) and download the binary with the access token **in a dedicated `media` job enqueued from the webhook**, before the reply job reads it. Files MUST be size-capped (`MEDIA_MAX_BYTES`, default 16 MB) and limited to a MIME allowlist. They are stored under `MEDIA_DIR` (local disk in the test profile) with a random filename, not the original one.

- **AC1:** Given an image message, when the media job runs, then the file exists on disk and `messages.media` records the path, MIME type, size and sha256.
- **AC2:** Given a 20 MB file, when downloading, then the download stops at the cap, the message is marked `media_rejected`, and the customer gets the fallback.
- **AC3:** Given a media URL that returns 404 (expired), when downloading, then the job fails permanently and the customer gets "please resend".

### HND-FR-08 — Reply waits for media
The reply job for a `wa_id` MUST NOT consume an inbound media message until its media job has finished (succeeded or failed permanently).

- **AC1:** Given text + image arrive together, when the reply job starts before the media job finishes, then it defers (re-enqueues with a delay) instead of replying without the image.

### HND-FR-09 — Voice notes to text
Downloaded audio MUST be transcribed by `STT_PROVIDER` (`google` = Gemini audio input, `whisper_local`). The transcript is stored in `messages.body` (with `messages.transcribed = true`) and then flows through the normal LLM path. Audio longer than `STT_MAX_SECONDS` (default 120) gets a fallback asking for a shorter message or text.

- **AC1:** Given a voice note, when processed, then the reply answers its spoken content. (manual)
- **AC2:** Given a transcription failure, when processed, then the customer gets "Sorry, I couldn't understand the voice note. Could you type it?"

### HND-FR-10 — Images
If the configured LLM supports image input, the image MUST be passed to the model with the customer's caption. Otherwise the bot MUST reply with a fallback asking for a text description. Documents and videos get a polite "can't open this" fallback in this spec.

- **AC1:** Given an image and a vision-capable model, when processed, then the model request includes the image part. (unit, on message assembly)

### HND-FR-11 — Other message types
`sticker` → treated as no-content (no reply unless other content is pending). `location` → acknowledged and passed to the model as text coordinates. `contacts` → "can't process contact cards". `reaction` → no reply (from 002). `unsupported` → generic fallback.

- **AC1:** One test per type confirms the behaviour above.

### HND-FR-12 — Media retention
Media files MUST be deleted after `MEDIA_RETENTION_DAYS` (default 30) by a scheduled job. Media MUST never be served from a public URL.

- **AC1:** Given a file older than the retention period, when the cleanup job runs, then the file is deleted and the DB row's `media.deleted_at` is set.

## Non-functional requirements

- **HND-NFR-01:** Media download starts within 5 s of the webhook (well within the URL's lifetime).
- **HND-NFR-02:** Admin endpoints are excluded from public docs and rate-limited (30 requests per minute per token).

## Out of scope

- Outbound media (sending images or files to customers).
- Multiple agents with assignment and presence (the Chatwoot/custom UI decision covers this).

## Open questions

- `[NEEDS CLARIFICATION: Inbox: Chatwoot or custom TanStack app?]` (blocks HND-FR-06 only)
- `[NEEDS CLARIFICATION: Agent working hours. Should handoff outside hours say "we'll reply at 9am"?]` Default: always the same acknowledgement.
