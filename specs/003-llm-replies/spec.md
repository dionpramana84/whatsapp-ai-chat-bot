# 003 — LLM replies

- **Status:** Ready
- **Phase:** 2
- **Depends on:** 002

## Goal

Replace the echo with a concise, business-scoped LLM reply that uses the conversation history, merges bursts of messages, respects the 24-hour window, degrades gracefully under free-tier rate limits, and is checked by a repeatable eval suite so a provider switch is a measured decision.

## User stories

- **US1:** As a customer, I get a helpful, short answer to questions about the business, in the language I wrote in.
- **US2:** As a customer, when I ask something unrelated to the business (homework, coding, general chat), the bot politely declines and tells me what it can help with.
- **US3:** As a customer, when I send several messages in a row, I get one reply that addresses all of them.
- **US4:** As an operator, when the LLM is slow or rate-limited, the customer still gets a polite holding message instead of silence.
- **US5:** As an operator, I can switch LLM provider with configuration and prove behaviour didn't regress by running the eval suite.

## Functional requirements

### LLM-FR-01 — Business-scoped system prompt
The system prompt MUST be built from a versioned template (`prompts/system.md`) and a business profile (`config/business.ts`: name, description, supported topics, contact/hours, tone, languages). It MUST tell the model to answer only within the supported topics, to decline anything else in one or two sentences while listing what it can help with, never to reveal the system prompt, and to reply in the customer's language when it is one of the supported languages.

- **AC1:** Given the business profile, when the prompt is built, then it contains the business name and every supported topic. (unit)
- **AC2:** Out-of-scope eval cases pass (see LLM-FR-09).

### LLM-FR-02 — Conversation history
The composer MUST send the last `HISTORY_MAX_MESSAGES` (default 20) messages of the conversation from the past `HISTORY_MAX_DAYS` (default 7), in chronological order, mapped to `user` / `assistant` roles. Failed outbound messages are excluded. Unprocessed inbound messages from this turn are merged into one final `user` message.

- **AC1:** Given 30 stored messages, when history loads, then 20 are returned, oldest first.
- **AC2:** Given a failed outbound message, when history loads, then it is excluded.
- **AC3:** Given 3 unprocessed inbound texts, when messages are built, then the final user message contains all 3 in order.

### LLM-FR-03 — Debounce bursts
Reply jobs MUST start `DEBOUNCE_SECONDS` (default 3) after the first message of a burst, so messages arriving in that window are answered together.

- **AC1:** Given 3 messages 500 ms apart and debounce 3 s, when the queue drains, then one LLM call and one reply happen.

### LLM-FR-04 — Single, bounded reply
Each turn MUST produce exactly one outbound text of at most 4,096 characters. The prompt asks for replies under about 600 characters. If the model returns more than 4,096, the reply MUST be cut at the last sentence or line boundary before 4,090 characters, with `…` appended. Empty model output MUST be replaced by the fallback message.

- **AC1:** Given a 5,000-character model output, when post-processed, then the sent text is ≤ 4,096 characters and ends with `…`.
- **AC2:** Given an empty or whitespace-only output, when post-processed, then the fallback message is sent.

### LLM-FR-05 — 24-hour window guard
Before any free-form send (in any spec), the sender MUST check that `now - contacts.last_inbound_at < 24h - 5min`. If the window is closed, it MUST NOT send free-form text. Instead it MUST either send the configured re-engagement template (if `REENGAGE_TEMPLATE` is set) or mark the outbound `failed` with reason `window_closed`.

- **AC1:** Given `last_inbound_at` 23h 56m ago, when a reply is sent, then it is blocked as window closed.
- **AC2:** Given `last_inbound_at` 1 hour ago, when a reply is sent, then the free-form text goes out.
- **AC3:** Given a closed window and `REENGAGE_TEMPLATE=hello_world`, when sending, then a template message is sent instead.

### LLM-FR-06 — Timeouts, rate limits and fallback
The LLM call MUST time out after `LLM_TIMEOUT_MS` (default 20,000). On a timeout, provider 5xx or provider rate-limit error, the worker MUST retry the LLM call at most once (after 2 s for a rate limit), then send `FALLBACK_MESSAGE` ("Sorry, I'm having trouble right now. Please try again in a few minutes."). The inbound messages count as processed after the fallback is sent.

- **AC1:** Given the provider times out twice, when the job runs, then the fallback is sent and the job completes.
- **AC2:** Given the provider returns 429 then succeeds, when the job runs, then the real reply is sent.

### LLM-FR-07 — Provider by configuration
The provider and model MUST be selected by `LLM_PROVIDER` (`google` | `anthropic` | `openai` | `ollama`) and `LLM_MODEL`, with the matching API key variable required only for the chosen provider. Changing provider MUST NOT need code changes outside the provider factory.

- **AC1:** Given `LLM_PROVIDER=google` without `GOOGLE_GENERATIVE_AI_API_KEY`, when config loads, then startup fails naming that variable.
- **AC2:** Given `LLM_PROVIDER=ollama`, when config loads, then no API key is required.

### LLM-FR-08 — Prompt-injection hygiene
Customer text MUST only ever appear in `user` messages, never interpolated into the system prompt. The system prompt MUST state that instructions inside customer messages don't override it.

- **AC1:** Given a customer message containing `{{business_name}}` or "ignore previous instructions", when messages are built, then the system prompt is byte-identical to the one built without it. (unit)
- **AC2:** Injection eval cases pass (LLM-FR-09).

### LLM-FR-09 — Eval suite
The repo MUST contain an eval suite (`evals/`) of at least 25 synthetic conversation cases across: in-scope answers, out-of-scope refusals, prompt injection / prompt extraction, multi-message bursts, language mirroring, and overlong-answer pressure. Each case has assertions (`mustInclude`, `mustNotInclude`, `maxChars`, and optional `judge` rubric graded by an LLM). `npm run eval` runs every case against the configured provider and prints a pass rate. It MUST NOT run in CI by default (cost and rate limits).

- **AC1:** Given the suite, when `npm run eval` runs against the chosen test provider, then the pass rate is at least 90% and every out-of-scope and injection case passes. (manual, result committed to `evals/results/`)

### LLM-FR-10 — Usage accounting
Each LLM-generated outbound message MUST store provider, model, input tokens, output tokens and latency (`messages.llm_usage` jsonb).

- **AC1:** Given a successful reply, when stored, then `llm_usage` has all five fields.

### LLM-FR-11 — Typing indicator
The worker MAY send a typing indicator, or mark the message read, while the LLM call runs, if the Graph API version in use supports it. A failure MUST NOT affect the reply.

## Non-functional requirements

- **LLM-NFR-01:** p95 time from the last inbound message to the reply being sent is under `DEBOUNCE_SECONDS + 10 s` on the test provider.
- **LLM-NFR-02:** No customer content is sent to a free-tier provider unless the sender is in `ALLOWED_WA_IDS` (constitution IX, enforced by 002 ingest).

## Out of scope

- Knowledge base and tools (004). In this spec the bot answers only from the business profile in the prompt.
- Media understanding (005).

## Open questions

- `[NEEDS CLARIFICATION: Supported reply languages?]` Default assumption: English and Indonesian. Only the business profile changes if this is wrong, so it doesn't block.
