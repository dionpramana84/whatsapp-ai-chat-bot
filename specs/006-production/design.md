# 006 — Production hardening + go-live: Design

## Hosting options (decision pending, PRD-FR-06)

| Option | Monthly cost (rough, verify) | Fit | Watch out |
|---|---|---|---|
| Hetzner/DO VPS + Docker Compose + managed Postgres | low | Cheapest always-on. Full control. | You own patching, backups and TLS (Caddy) |
| Fly.io (2 machines + Postgres) | low–medium | Easy `ROLE` split. Release command for migrations. | Check pgvector availability on the chosen Postgres offering |
| Railway | low–medium | Simplest DX. Postgres template with pgvector. | Usage-based billing. Set spend limits. |
| Oracle Always Free + self-managed Postgres | $0 | Viable for a small business | Reclaim policy for idle instances, ARM images, no managed backups |

My recommendation: a VPS + managed Postgres for a single business. Fly or Railway if nobody wants to own a server. All of these are just "a container with `ROLE`", so switching later is cheap.

## Graph error policy table

Single source of truth in `src/whatsapp/errors.ts`:

| Code / status | Class | Action |
|---|---|---|
| 429, `130429` | retryable | exponential backoff with jitter, base 2 s |
| `131056` (pair rate limit) | retryable | `startAfter ≥ 6 s` for this `wa_id` |
| 5xx, network, timeout | retryable | backoff |
| `131047` (outside window) | permanent | mark failed `window_closed` (shouldn't happen with the 003 guard, so alert if it does) |
| `131026` (undeliverable) | permanent | mark failed. Don't retry. |
| `190` (token invalid/expired) | fatal | alert + `boss.offWork('reply')` until an admin resumes |
| other 4xx | permanent | mark failed + Sentry |

Codes are from memory and **must be checked** against Meta's error code reference before implementation.

## Observability

- **Logs:** pino JSON to stdout, shipped by the host. Correlation fields: `wa_id` (hashed in prod logs: `sha256(wa_id + LOG_SALT)[:12]`), `wamid`, `job_id`.
- **Errors:** Sentry SDK in both roles. `beforeSend` scrubs anything matching token-like patterns and drops `request.data`.
- **Metrics:** at this scale, no Prometheus. A `job_runs` view over pg-boss tables + `messages.llm_usage` + `messages` categories is enough. `GET /admin/metrics` aggregates with SQL.
- **Alert delivery:** a scheduled `maintenance` job evaluates thresholds every minute and posts to `ALERT_WEBHOOK_URL` (Slack/Discord). That's free, with no extra service. Swap for Sentry alerts or Grafana once the team wants dashboards.

## Cost model

```ts
// config/pricing.ts: operator-maintained, values from provider pages at go-live
export const llmPrices = { "<model-id>": { inputPerMTok: 0, outputPerMTok: 0 } };
export const metaRates = { service: 0, utility: 0, marketing: 0, authentication: 0 }; // per message, per market
```

The cost is computed when the message is written and stored in `messages.cost` (jsonb `{llm, meta, currency}`), so later price changes don't rewrite history.

## Opt-in

- Sources: website checkbox, in-chat "Reply YES to receive updates", in-store QR. Each sets `opt_in_source`.
- In-chat opt-in is a keyword flow handled before the LLM, like the handoff keywords.
- Opt-out keywords are checked first of all. They override the pause, the handoff and everything else.

## Load test

- k6 script signs payloads with the staging app secret (passed by env, never committed).
- Graph API mock: a small Hono app returning a random `wamid` with a configurable latency or error rate. LLM mock: the same idea, deterministic text.
- Assertions run in SQL after the test: `count(*) per inbound wamid = 1`, `count(outbound) per wa_id burst = 1`, and ordering checked by `meta_timestamp` vs `created_at`.

## Deployment pipeline

```
push to main → CI (001) → build image → push to registry → release: npm run db:migrate
→ deploy web (rolling) → deploy worker (rolling, graceful stop)
```

Migrations must be backward-compatible with the previous version (expand → migrate → contract), because the worker and web roll separately.

## Constitution check

| Article | How this design complies |
|---|---|
| IV | Load test asserts no duplicates under 3× redelivery |
| V | Rotation runbook, Sentry scrubbing, hashed `wa_id` in prod logs |
| VIII | Paid items only here, listed in the go-live checklist |
| IX | Checklist item 7 ties removing the allowlist to every provider being on a paid tier |
| X | Opt-in, templates, error-code policy |
