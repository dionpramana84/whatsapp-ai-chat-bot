# Constitution

These rules can't be broken. Every spec, design, task and pull request must comply. Each `design.md` has a "Constitution check" section that lists any article it touches. Changing this file needs a recorded amendment at the bottom.

## I. Official channel only

The bot talks to WhatsApp only through Meta's Cloud API. Unofficial clients (Baileys, whatsapp-web.js, browser automation) are forbidden: they violate Meta's terms and get numbers banned.

## II. Business-scoped bot

The bot serves one business's customer use cases. Since 15 Jan 2026, Meta's WhatsApp Business Solution Terms ban general-purpose AI assistants. Scope is enforced in **both** the system prompt and the tool set. Out-of-scope requests get a short, polite refusal that points back to what the bot can do. The eval suite (003) includes out-of-scope cases and must pass before any provider or prompt change ships.

## III. The sender is the identity

The customer's identity comes **only** from the `wa_id` in a signature-verified webhook. Tools receive it from the server-side job context, never from message text or LLM-generated arguments. No tool accepts a customer, order or account identifier that could let one sender reach another sender's data.

## IV. Idempotent by default

Meta retries webhooks and pg-boss retries jobs. Every webhook handler and job handler must be safe to run more than once with the same input. `wamid` is the dedupe key for messages. Duplicate deliveries are expected, not exceptional.

## V. Secrets stay server-side

Access tokens, app secrets and API keys live only in environment variables or a secrets manager. They are never committed, never logged, never sent to the client, and never put in an LLM prompt. `.env` is git-ignored. The logger redacts known secret fields and `Authorization` headers. Production uses a system-user token, never the 24-hour dashboard token.

## VI. Verify before trust

Every `POST /webhook` verifies `X-Hub-Signature-256` (HMAC-SHA256 with the app secret) over the **raw request bytes**, with a constant-time comparison, before parsing or persisting anything. Every payload is validated with a schema before use. LLM output is untrusted input to anything it calls.

## VII. Spec first, test first

No production code without a requirement ID. Each acceptance criterion has an automated test unless it is tagged `(manual)`, in which case the manual check is written down in the task. If the code needs to differ from the spec, the spec changes first.

## VIII. $0 until go-live

Everything before 006's go-live tasks must run with zero spend, using the "$0 testing profile" in `PLAN.md`. Adding a paid dependency before then needs an amendment.

## IX. No real customer data on free tiers

Free LLM, STT and embedding tiers may use inputs for training. While any free tier is configured, the bot only processes messages from numbers in `ALLOWED_WA_IDS`. Test conversations use synthetic data.

## X. Platform constraints are requirements

The 24-hour service window, template rules, opt-in, the 4,096-character text limit, rate limits and media URL expiry are treated as functional requirements with tests, not as later polish.

## XI. Simple over clever

- TypeScript strict, no `any`. Types are inferred from zod and Drizzle schemas rather than declared twice.
- No abstraction until there is a second use case.
- Comments explain *why*, never *what*.
- Dependencies are pinned. A new dependency needs a one-line justification in the design.

## Amendments

| Date | Article | Change | Reason |
|---|---|---|---|
| — | — | — | — |
