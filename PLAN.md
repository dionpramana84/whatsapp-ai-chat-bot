# WhatsApp AI Chatbot — Implementation Plan

A WhatsApp chatbot for a single business's customer flows (support, orders, bookings), built on the official WhatsApp Cloud API with an LLM and retrieval over the business's own knowledge base.

> **Scope constraint:** Since 15 Jan 2026, Meta's WhatsApp Business Solution Terms prohibit general-purpose AI assistants. This bot stays scoped to one business's use cases. The system prompt and tool set have to enforce that.

> **How this plan is executed:** Spec-Driven Development. This file is the high-level roadmap. The executable specs, designs and task lists live in [`specs/`](specs/README.md), governed by [`specs/constitution.md`](specs/constitution.md). When this file and a spec disagree, the spec wins. Update this file afterwards.

## Stack

| Layer | Choice | Notes |
|---|---|---|
| WhatsApp channel | Meta Cloud API (direct) | No BSP markup. No unofficial libraries (Baileys, whatsapp-web.js): they break the ToS and get numbers banned. |
| Language | TypeScript (Node 24 LTS) | Strict mode, inference over annotation. Node 20 reached end-of-life in April 2026. |
| Webhook server | Hono | Small and fast. Move to TanStack Start server routes only if the admin UI ends up in the same app. |
| Queue | pg-boss | Reuses Postgres, so there's no Redis to run. |
| Database | Postgres + pgvector | Holds state, message history, and embeddings. Drizzle ORM for schema, migrations and inferred types. |
| LLM | Vercel AI SDK + provider (Gemini free tier for testing; Claude / OpenAI / Gemini for production) | Tool calling, and providers can be swapped. |
| Knowledge base admin | Markdown files in `kb/` for testing. Payload CMS (optional, later) | Non-devs edit FAQs and products. An `afterChange` hook re-embeds documents. |
| Human inbox | Admin API for testing. Chatwoot or a custom TanStack app later | |
| Hosting | Local + ngrok for testing. VPS / Fly / Railway for production | Long-running worker process. No pure serverless. |

## $0 testing profile

Everything up to the go-live phase runs with zero spend. Constitution article VIII makes this binding.

| Layer | Free option | Limits to design around |
|---|---|---|
| WhatsApp | Meta **test phone number** | Up to 5 verified recipient numbers. Only the `hello_world` template is pre-approved. Fixed display name. |
| Hosting | Local machine, Docker Compose (Postgres + pgvector) | Runs only while the machine is on. |
| Public HTTPS | **ngrok free static domain** | A cloudflared quick tunnel changes its URL on every restart, which means re-registering the webhook each time. |
| LLM | Gemini API free tier (Flash model) | Rate limits change often, so check the current numbers. **Free-tier inputs may be used for training: no real customer data.** |
| Embeddings | Gemini embedding model at 768 dimensions, or Ollama `nomic-embed-text` | Fix the dimension before the first ingest. Changing it means re-embedding everything. |
| Speech-to-text | Gemini audio input (free tier) or local whisper.cpp | Same data caveat. |
| CI | GitHub Actions | Free for public repos. Private repos get a monthly minute allowance. |
| Errors | Sentry free developer plan | |

Avoid these for this architecture: **Render free** (sleeps, which kills the worker) and **Neon free** (pg-boss polling keeps the database awake and uses up the compute hours). For an always-on $0 host, Oracle Cloud Always Free (ARM VM) is the only realistic option, and it needs a card to verify your identity.

Verify on Meta's pricing page whether messages sent from the test number are billed.

## Architecture

```
Meta ──POST──▶ /webhook (verify HMAC on raw body → persist inbound idempotently → enqueue → 200 OK)
                   │
                   ▼
              pg-boss "reply" queue (singletonKey = wa_id, debounce via startAfter)
                   │
                   ▼
              reply worker (one active job per wa_id)
                   │ load unprocessed inbound + history + RAG context
                   │ LLM + tools (scoped to verified wa_id)
                   │ persist outbound as pending + mark inbound processed (one transaction)
                   ▼
              Graph API send ──▶ update outbound with wamid/status
```

The detailed design is in [`specs/002-echo-bot/design.md`](specs/002-echo-bot/design.md).

## Data model (initial)

- `contacts`: `wa_id` (PK), `name`, `bot_paused` (bool, for human handoff), `last_inbound_at` (24-hour window), `created_at`, `updated_at`
- `messages`: `id`, `wamid` (unique, null until an outbound message is accepted by Meta), `wa_id`, `direction` (in/out), `type`, `body`, `status`, `error`, `processed_at` (inbound only), `meta_timestamp`, `created_at`
- `kb_chunks`: `id`, `source`, `content`, `content_hash`, `embedding vector(768)`, `updated_at`

Idempotency comes from the unique constraint on `messages.wamid`. A separate `processed_events` table isn't needed.

## Phases → specs

| Phase | Spec | Outcome |
|---|---|---|
| 0: Setup | [`001-foundation`](specs/001-foundation/spec.md) | Repo scaffold, local stack, Meta test app, tunnel |
| 1: Echo bot | [`002-echo-bot`](specs/002-echo-bot/spec.md) | Signed webhook → queue → idempotent reply pipeline |
| 2: LLM replies | [`003-llm-replies`](specs/003-llm-replies/spec.md) | Scoped LLM replies, history, debounce, 24-hour window, evals |
| 3: RAG + tools | [`004-rag-tools`](specs/004-rag-tools/spec.md) | Knowledge-base retrieval, sender-scoped tools |
| 4: Handoff + media | [`005-handoff-media`](specs/005-handoff-media/spec.md) | Human handoff, voice notes, images |
| 5: Production + go-live | [`006-production`](specs/006-production/spec.md) | Hardening, templates, opt-in, load test, go-live checklist |

Each spec folder has `spec.md` (what and why), `design.md` (how) and `tasks.md` (ordered, traceable tasks).

## Platform rules to design around

| Rule | Impact |
|---|---|
| General-purpose AI bots banned (since 15 Jan 2026) | Keep scope to the business, in both prompt and tools |
| 24-hour customer service window | Free-form replies only within 24 h of the user's last message. Templates outside it. |
| Opt-in required for business-initiated messages | Record opt-in (source and time) before sending any template |
| Per-message pricing (since 1 Jul 2025) | Budget per message |
| Service-reply pricing changes announced for 1 Oct 2026 | **That date has passed: confirm the current rate card before 006.** Avoid multi-bubble replies either way. |
| 4,096-character text limit | Truncate or summarize |
| Webhook signature (`X-Hub-Signature-256`) is over the raw body | In Hono, read `arrayBuffer()` before any JSON parsing |
| Meta retries webhooks that don't get a 2xx | Handlers must be idempotent. Ack unknown-but-signed events with 200. |
| Media URLs expire within minutes | Download immediately on receipt |

## Failure modes checklist

| Risk | Severity | Mitigation | Spec |
|---|---|---|---|
| Concurrent messages from one user | High | Per-`wa_id` singleton job + debounce | 002, 003 |
| Webhook retries → duplicate replies | High | Unique `wamid`, idempotent worker | 002 |
| Insert succeeded, enqueue failed → lost reply | High | Enqueue whenever unprocessed inbound exists, even on conflict | 002 |
| Crash between LLM call and send | Medium | Persist pending outbound first. Retry sends it instead of regenerating. | 002 |
| Prompt injection via tools | High | Tools scoped to the verified sender | 004 |
| Leaked access token | High | Server-side only, system-user token, rotate, redact logs | 001, 006 |
| Real customer data on a free LLM tier | High | `ALLOWED_WA_IDS` allowlist, test number only | 002 |
| Free-tier rate limit hit | Medium | Timeout + fallback message | 003 |
| Voice notes / images | Medium | STT / vision, or a fallback | 005 |
| Bot talks over a human agent | Medium | `bot_paused` flag | 005 |
| Overlong replies | Low | Length cap | 003 |

## Open questions

These block parts of 004–006 and are marked `[NEEDS CLARIFICATION]` in the specs.

- What is the bot's primary job: support, sales, bookings, or something else?
- Which LLM provider for production?
- Is there an existing system (store, booking tool, CRM) the tools need to call?
- Human inbox: Chatwoot (self-hosted, heavier) or a custom TanStack app?
- Production hosting budget and target?
- Which languages should the bot reply in?
