# WhatsApp AI Chatbot — Implementation Plan

A WhatsApp chatbot for a single business's customer flows (support, orders, bookings), built on the official WhatsApp Cloud API with an LLM and retrieval over the business's own knowledge base.

> **Scope constraint:** Since 15 Jan 2026, Meta's WhatsApp Business Solution Terms prohibit general-purpose AI assistants. This bot stays scoped to one business's use cases. The system prompt and tool set have to enforce that.

## Stack

| Layer | Choice | Notes |
|---|---|---|
| WhatsApp channel | Meta Cloud API (direct) | No BSP markup. No unofficial libraries (Baileys, whatsapp-web.js): they break the ToS and get numbers banned. |
| Language | TypeScript (Node 20+) | Strict mode, inference over annotation. |
| Webhook server | Hono | Small and fast. Move to TanStack Start server routes only if the admin UI ends up in the same app. |
| Queue | pg-boss | Reuses Postgres, so there's no Redis to run. |
| Database | Postgres + pgvector | Holds state, message history, and embeddings. |
| LLM | Vercel AI SDK + provider (Claude / OpenAI / Gemini) | Tool calling, and providers can be swapped. |
| Knowledge base admin | Payload CMS (optional, phase 3) | Non-devs edit FAQs and products. An `afterChange` hook re-embeds documents. |
| Human inbox | Chatwoot or a custom TanStack app (phase 4) | |
| Hosting | VPS / Fly / Railway | Long-running worker process. No pure serverless. |

## Architecture

```
Meta ──POST──▶ /webhook (verify HMAC, enqueue, 200 OK)
                   │
                   ▼
              pg-boss queue ──▶ worker (singleton per wa_id)
                                   │ dedupe on wamid
                                   │ load history + RAG context
                                   │ LLM + tools
                                   ▼
                          Graph API send ──▶ store outbound message
```

## Data model (initial)

- `contacts`: `wa_id` (PK), `name`, `created_at`, `bot_paused` (bool, for human handoff)
- `messages`: `id`, `wamid` (unique), `wa_id`, `direction` (in/out), `type`, `body`, `status`, `created_at`
- `kb_documents`: `id`, `source`, `content`, `embedding vector(n)`, `updated_at`
- `processed_events`: `wamid` (PK), for idempotency (or a unique constraint on `messages.wamid`)

## Phases

### Phase 0: Setup
- [ ] Meta Business account, WhatsApp Business app, test number
- [ ] Permanent **system-user** access token (not the 24-hour dashboard token), kept in a secrets manager or env only
- [ ] Repo scaffold: TS strict, ESLint, Prettier, Vitest, Docker Compose (Postgres + pgvector)
- [ ] Public HTTPS tunnel for local webhook testing (cloudflared / ngrok)

### Phase 1: Echo bot (proves the pipe)
- [ ] `GET /webhook`: verification handshake (`hub.mode`, `hub.verify_token`, `hub.challenge`)
- [ ] `POST /webhook`: verify the `X-Hub-Signature-256` HMAC against the raw body, enqueue the job, return 200 immediately
- [ ] Worker: dedupe on `wamid`, store the inbound message, send an echo reply through the Graph API
- [ ] Handle status webhooks (sent/delivered/read/failed) separately from messages

### Phase 2: LLM replies
- [ ] System prompt scoped to the business's use cases (policy compliance)
- [ ] Conversation history window (last N messages per `wa_id`)
- [ ] Serialize jobs per `wa_id` + a 2–3 s debounce to merge bursts of messages
- [ ] One concise reply per turn (cost control, since service messages are billed). Cap at 4,096 characters.
- [ ] 24-hour window check before any send. Fall back to an approved template outside the window.

### Phase 3: RAG + tools
- [ ] Ingest the knowledge base into pgvector
- [ ] Retrieval step before the LLM call
- [ ] Business tools (e.g. order lookup, booking). **Every tool is scoped to the sender's verified `wa_id`, never to IDs the user supplies.**
- [ ] Optional: Payload CMS as the KB admin, with an `afterChange` hook that re-embeds documents

### Phase 4: Human handoff + media
- [ ] `handoff` tool sets `bot_paused`. The bot stays silent while an agent handles the conversation.
- [ ] Agent inbox (Chatwoot integration or custom UI)
- [ ] Voice notes → speech-to-text. Images → vision model or a polite fallback.
- [ ] Download media promptly (Meta media URLs expire)

### Phase 5: Production hardening
- [ ] Retries with backoff on Graph API 5xx and rate limits
- [ ] LLM timeout + fallback message
- [ ] Structured logging, error tracking, per-conversation cost metrics
- [ ] Message templates submitted and approved for re-engagement flows
- [ ] Load test with bursty, concurrent senders

## Platform rules to design around

| Rule | Impact |
|---|---|
| General-purpose AI bots banned (since 15 Jan 2026) | Keep scope to the business, in both prompt and tools |
| 24-hour customer service window | Free-form replies only within 24 h of the user's last message. Templates outside it. |
| Per-message pricing (since 1 Jul 2025) | Budget per message |
| Service replies reportedly charged from 1 Oct 2026 (after 1,000 free/number/month) | **Verify on Meta's rate card.** Avoid multi-bubble replies. |
| 4,096-character text limit | Truncate or summarize |

## Failure modes checklist

| Risk | Severity | Mitigation |
|---|---|---|
| Concurrent messages from one user | High | Per-`wa_id` serialization + debounce |
| Webhook retries → duplicate replies | High | Unique `wamid`, idempotent worker |
| Prompt injection via tools | High | Tools scoped to the verified sender |
| Leaked access token | High | Server-side only, system-user token, rotate |
| Voice notes / images | Medium | STT / vision, or a fallback |
| Bot talks over a human agent | Medium | `bot_paused` flag |
| Overlong replies | Low | Length cap |

## Open questions

- What is the bot's primary job: support, sales, bookings, or something else?
- Which LLM provider?
- Is there an existing system (store, booking tool, CRM) the tools need to call?
