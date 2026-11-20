# 001 — Foundation: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

## Group A — Scaffold

- [ ] FND-T01 Initialize the Node 24 + TypeScript strict ESM project
  - Refs: FND-FR-01
  - Files: package.json, tsconfig.json, .nvmrc, .gitignore, src/main.ts
  - Done when: `npm run typecheck` passes. `tsconfig` has `strict`, `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`. `.gitignore` covers `node_modules`, `dist`, `.env*` and keeps `!.env.example`.

- [ ] FND-T02 [P] Add ESLint (typescript-eslint strict-type-checked) + Prettier
  - Refs: FND-FR-01
  - Files: eslint.config.js, .prettierrc, package.json
  - Depends: FND-T01
  - Done when: `npm run lint` and `npm run format:check` pass. A fixture with an explicit `any` fails lint (FND-FR-01.AC2, checked once by hand and recorded in the PR).

- [ ] FND-T03 [P] Add Vitest with a smoke test
  - Refs: FND-FR-01
  - Files: vitest.config.ts, test/smoke.test.ts
  - Depends: FND-T01
  - Done when: `npm test` runs and passes (FND-FR-01.AC1).

## Group B — Config and logging

- [ ] FND-T04 Write config tests (missing var, secret not echoed, happy path)
  - Refs: FND-FR-04
  - Files: test/config.test.ts
  - Depends: FND-T03
  - Done when: the tests exist and fail.

- [ ] FND-T05 Implement `src/config.ts` with a zod schema and safe error formatting
  - Refs: FND-FR-04
  - Files: src/config.ts
  - Depends: FND-T04
  - Done when: FND-FR-04.AC1–AC3 pass. The exported `Config` type is inferred from the schema.

- [ ] FND-T06 Add `.env.example` and a key-parity test
  - Refs: FND-FR-05
  - Files: .env.example, test/env-example.test.ts
  - Depends: FND-T05
  - Done when: FND-FR-05.AC1 passes.

- [ ] FND-T07 [P] Implement the pino logger with redaction + test
  - Refs: FND-FR-09
  - Files: src/logger.ts, test/logger.test.ts
  - Depends: FND-T03
  - Done when: FND-FR-09.AC1 passes.

## Group C — Database

- [ ] FND-T08 Add Docker Compose with pgvector Postgres
  - Refs: FND-FR-03
  - Files: docker-compose.yml
  - Done when: `docker compose up -d db` becomes healthy and `CREATE EXTENSION vector` works (FND-FR-03.AC1, manual, recorded in `docs/local-dev.md`).

- [ ] FND-T09 Add the Drizzle client, config and initial migration (vector extension, `contacts`, `messages` per 002 design)
  - Refs: FND-FR-06
  - Files: src/db/client.ts, src/db/schema.ts, drizzle.config.ts, drizzle/0000_*.sql, package.json (`db:generate`, `db:migrate`)
  - Depends: FND-T05, FND-T08
  - Done when: FND-FR-06.AC1 passes in `test/db/migrate.test.ts`. The migration starts with `CREATE EXTENSION IF NOT EXISTS vector`.
  - Note: the table definitions come from `specs/002-echo-bot/design.md#data-model`. Defining them here keeps a single initial migration.

- [ ] FND-T10 Add integration-test DB helpers (migrate once, truncate between tests)
  - Refs: FND-FR-06
  - Files: test/setup/db.ts, vitest.config.ts
  - Depends: FND-T09
  - Done when: integration tests can opt in to a clean database. Tests run against a `_test` database, never the dev database.

## Group D — App shell

- [ ] FND-T11 Write `/healthz` tests (ok, db down → 503 without stack)
  - Refs: FND-FR-07
  - Files: test/http/healthz.test.ts
  - Depends: FND-T10
  - Done when: the tests exist and fail.

- [ ] FND-T12 Implement the `createApp(deps)` Hono factory with `/healthz`
  - Refs: FND-FR-07
  - Files: src/http/app.ts
  - Depends: FND-T11
  - Done when: FND-FR-07.AC1–AC2 pass. Unhandled errors return a generic 500 JSON and are logged.

- [ ] FND-T13 Add the pg-boss factory (start, create queues, stop)
  - Refs: FND-FR-08
  - Files: src/queue/boss.ts
  - Depends: FND-T09
  - Done when: the boss starts against the test DB in an integration test and stops cleanly.

- [ ] FND-T14 Implement `main.ts` with roles and graceful shutdown + tests
  - Refs: FND-FR-08
  - Files: src/main.ts, test/main.test.ts
  - Depends: FND-T12, FND-T13
  - Done when: FND-FR-08.AC1–AC2 pass (spawn the process in the test and send SIGTERM).

## Group E — CI and docs

- [ ] FND-T15 [P] Add the GitHub Actions CI workflow with a pgvector service
  - Refs: FND-FR-02
  - Files: .github/workflows/ci.yml
  - Depends: FND-T10
  - Done when: CI is green on the PR. A deliberately failing test turns it red (FND-FR-02.AC1, manual).

- [ ] FND-T16 [P] Write `docs/meta-setup.md`
  - Refs: FND-FR-10
  - Files: docs/meta-setup.md
  - Done when: a second person (or a fresh account) completes setup by following it (manual). The doc warns against the 24-hour dashboard token.

- [ ] FND-T17 [P] Write `docs/local-dev.md` (compose, env, migrate, run, ngrok static domain, cloudflared caveat)
  - Refs: FND-FR-11, FND-NFR-02
  - Files: docs/local-dev.md, README.md
  - Done when: FND-FR-11.AC1 is verified manually. README links to `PLAN.md`, `specs/` and both docs.

## Exit criteria

- Every box ticked. Every AC verified (automated or recorded manual check).
- CI green on `main`.
- Spec status set to `Done` in `specs/README.md`.
