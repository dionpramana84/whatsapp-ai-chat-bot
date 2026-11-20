# 001 — Foundation: Design

## Decisions

| Concern | Choice | Why | Alternative and when it wins |
|---|---|---|---|
| Runtime | Node 24 LTS, ESM | Current LTS. Node 20 is end-of-life. | Bun: faster dev loop, but pg-boss and the Postgres drivers are tested on Node |
| Dev runner | `tsx watch` | Zero-config TS execution | `node --experimental-strip-types`: fewer deps, but no path aliases or enums |
| Build | `tsc` to `dist/` | One tool, no bundler needed for a server | tsup/esbuild if cold start matters (it doesn't here) |
| HTTP | Hono + `@hono/node-server` | Small, typed, raw-body access is easy | Fastify: bigger ecosystem, heavier |
| DB driver + ORM | `postgres` (postgres.js) + Drizzle ORM / drizzle-kit | Types inferred from the schema, SQL-like API, pgvector column support | Kysely: equally good. Pick it if you prefer a pure query builder. |
| Queue | pg-boss | Uses the same Postgres, so there's nothing else to run | BullMQ: needs Redis |
| Validation | zod | Config, webhook payloads, tool schemas; types inferred | valibot: smaller bundle (irrelevant on a server) |
| Logging | pino | JSON, fast, built-in `redact` paths | — |
| Tests | Vitest | Fast, ESM-native | — |
| Lint/format | ESLint (typescript-eslint, strict-type-checked) + Prettier | Standard | Biome: one tool, but fewer type-aware rules |
| DB image | `pgvector/pgvector:pg17` | Official image with the extension already built | — |

**Version check before you start:** pin exact versions in `package.json`. pg-boss (v10+ changed the API: queues must be created, handlers get arrays of jobs), Drizzle (pgvector helpers) and Hono move quickly. Check their current docs.

## Repository layout

```
.
├── docker-compose.yml
├── .env.example
├── drizzle.config.ts
├── drizzle/                     # generated SQL migrations (committed)
├── docs/
│   ├── meta-setup.md
│   └── local-dev.md
├── src/
│   ├── main.ts                  # entry: reads ROLE, wires web/worker, shutdown
│   ├── config.ts                # zod env schema → frozen config
│   ├── logger.ts                # pino with redact paths
│   ├── db/
│   │   ├── client.ts            # postgres.js pool + drizzle instance
│   │   └── schema.ts            # tables (filled in by 002)
│   ├── http/
│   │   └── app.ts               # Hono app factory: createApp(deps)
│   └── queue/
│       └── boss.ts              # pg-boss factory + queue creation
├── test/
│   ├── setup/db.ts              # migrate + truncate helpers for integration tests
│   └── …
└── .github/workflows/ci.yml
```

`createApp(deps)` takes its dependencies (db, boss, config, logger, fetch) as arguments, so tests build an app with fakes without starting a server.

## Configuration schema

| Variable | Required | Default | Notes |
|---|---|---|---|
| `NODE_ENV` | no | `development` | `development` \| `test` \| `production` |
| `ROLE` | no | `all` | `web` \| `worker` \| `all` |
| `PORT` | no | `3000` | integer 1–65535 |
| `LOG_LEVEL` | no | `info` | pino levels |
| `DATABASE_URL` | yes | — | `postgres://…` |
| `WHATSAPP_ACCESS_TOKEN` | yes | — | system-user token, secret |
| `WHATSAPP_APP_SECRET` | yes | — | for HMAC, secret |
| `WHATSAPP_VERIFY_TOKEN` | yes | — | random string ≥ 16 chars you choose |
| `WHATSAPP_PHONE_NUMBER_ID` | yes | — | numeric string |
| `GRAPH_API_VERSION` | no | pinned, e.g. `v23.0` | check the latest version on Meta's changelog |
| `ALLOWED_WA_IDS` | no | empty | comma-separated, used by 002 |

Later specs add their own variables. Each one adds its keys to the schema and to `.env.example` in the same task (FND-FR-05.AC1 keeps them in sync).

Error formatting: map `ZodError.issues` to `"<VAR>: <message>"` lines. zod issue messages don't include the received value by default. A test (FND-FR-04.AC2) makes sure that stays true.

## Logging redaction paths

```
*.accessToken, *.appSecret, *.apiKey, *.token,
headers.authorization, req.headers.authorization,
headers["x-hub-signature-256"]
```

## Docker Compose

- `db`: `pgvector/pgvector:pg17`, port `5432`, volume `pgdata`, healthcheck `pg_isready`.
- The app is **not** containerized for local dev. It runs with `tsx watch` on the host for a fast loop. A `Dockerfile` arrives in 006.

## Shutdown sequence

```
SIGTERM → server.close() (stop accepting) → boss.stop({ graceful: true, timeout: 10_000 })
        → sql.end({ timeout: 5 }) → process.exit(0)
```

A second signal forces `process.exit(1)`.

## CI

GitHub Actions job with a `pgvector/pgvector:pg17` service container. Steps: `npm ci` → `typecheck` → `lint` → `format:check` → `db:migrate` → `test`. Cache npm.

## Constitution check

| Article | How this design complies |
|---|---|
| V Secrets | Config never prints values. pino redacts. `.env` is ignored. |
| VII Test first | Vitest + CI from day one |
| VIII $0 | Local Docker + ngrok free + GitHub Actions |
| XI Simple | No bundler, no DI container. Plain factory functions. |
