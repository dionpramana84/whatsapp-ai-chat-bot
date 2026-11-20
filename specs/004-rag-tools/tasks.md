# 004 — RAG + tools: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

> Groups A–D can start now. Group E is blocked by `[NEEDS CLARIFICATION]` in RAG-FR-08.

## Group A — Storage and ingest

- [ ] RAG-T01 Migration: `kb_chunks` (vector(768), HNSW cosine) and `orders` fixture table
  - Refs: RAG-FR-02, RAG-FR-07
  - Files: src/db/schema.ts, drizzle/000X_*.sql
  - Done when: the migration applies. `EXPLAIN` of a top-k query shows the HNSW index (note it in the PR).

- [ ] RAG-T02 [P] Markdown chunker (heading split, size cap, overlap, stable anchors): tests, then implementation
  - Refs: RAG-FR-01
  - Files: src/kb/chunk.ts, test/kb/chunk.test.ts
  - Done when: chunks never cross files, anchors are stable across runs, and no chunk is over the cap.

- [ ] RAG-T03 [P] Embedding config + `embedTexts` (batching, 429 backoff) with an injectable embed function
  - Refs: RAG-FR-02
  - Files: src/config.ts, .env.example, src/kb/embed.ts, test/kb/embed.test.ts
  - Done when: batching and retry are unit-tested with a fake. `EMBEDDING_*` and `RAG_*` vars are in the config and `.env.example`.

- [ ] RAG-T04 Ingest CLI tests (idempotent, single-section change, deleted file, model mismatch → rebuild)
  - Refs: RAG-FR-01, RAG-FR-02
  - Files: test/kb/ingest.test.ts
  - Depends: RAG-T01, RAG-T02, RAG-T03
  - Done when: the tests exist and fail.

- [ ] RAG-T05 Implement `npm run kb:ingest [--rebuild]`
  - Refs: RAG-FR-01, RAG-FR-02
  - Files: src/kb/ingest.ts, scripts/kb-ingest.ts, package.json
  - Depends: RAG-T04
  - Done when: RAG-FR-01.AC1–AC3 pass.

- [ ] RAG-T06 [P] Seed a synthetic test knowledge base (8–15 files: hours, shipping, returns, payment, products, contact)
  - Refs: RAG-FR-01
  - Files: kb/*.md
  - Done when: the content is synthetic and covers the 003 in-scope eval topics.

## Group B — Retrieval

- [ ] RAG-T07 Retrieval query + model-mismatch startup check: tests, then implementation
  - Refs: RAG-FR-02, RAG-FR-03
  - Files: src/kb/retrieve.ts, src/main.ts, test/kb/retrieve.test.ts
  - Depends: RAG-T05
  - Done when: RAG-FR-03.AC1 and RAG-FR-02.AC1 pass (integration with deterministic fake embeddings).

- [ ] RAG-T08 Reference-block assembly in `buildMessages` + injection-isolation test
  - Refs: RAG-FR-03
  - Files: src/llm/build-messages.ts, prompts/system.md, test/llm/build-messages.test.ts
  - Depends: RAG-T07
  - Done when: RAG-FR-03.AC2 passes. The 003 tests still pass.

## Group C — Tool framework

- [ ] RAG-T09 `withToolGuard` (timeout, error mapping, logging): tests, then implementation
  - Refs: RAG-FR-06, RAG-FR-09
  - Files: src/tools/guard.ts, test/tools/guard.test.ts
  - Done when: RAG-FR-06.AC2 passes.

- [ ] RAG-T10 `lookup_my_orders` + `findOrdersForSender` + seed script: tests, then implementation
  - Refs: RAG-FR-05, RAG-FR-07
  - Files: src/tools/index.ts, src/tools/orders.ts, scripts/seed-orders.ts, test/tools/orders.test.ts
  - Depends: RAG-T01, RAG-T09
  - Done when: RAG-FR-05.AC1 passes (cross-sender → identical not-found).

- [ ] RAG-T11 [P] Static tool-schema identity-key lint test
  - Refs: RAG-FR-05
  - Files: test/tools/schema-lint.test.ts
  - Depends: RAG-T10
  - Done when: RAG-FR-05.AC2 passes and fails when a `phone` field is added on purpose.

## Group D — Composer integration

- [ ] RAG-T12 Wire retrieval + tools into `composeLlm` with the `TOOL_MAX_STEPS` stop condition
  - Refs: RAG-FR-03, RAG-FR-06, RAG-FR-07, RAG-FR-09
  - Files: src/llm/compose-llm.ts, src/worker/reply.ts, test/llm/compose-llm.test.ts
  - Depends: RAG-T08, RAG-T10
  - Done when: RAG-FR-06.AC1, RAG-FR-07.AC1 and RAG-FR-09.AC1 pass with a mock model. Job `expireInSeconds` is reviewed against the worst-case step time.

- [ ] RAG-T13 Add grounding and cross-sender eval cases. Run evals and tune `RAG_MIN_SCORE`.
  - Refs: RAG-FR-04, RAG-FR-05
  - Files: evals/cases/grounding-*.yaml, evals/cases/tool-*.yaml, evals/results/*.json
  - Depends: RAG-T12
  - Done when: RAG-FR-04.AC1 passes. The 003 eval thresholds still hold.

- [ ] RAG-T14 Live test on the test number
  - Refs: US1–US4
  - Depends: RAG-T13
  - Done when (record in the PR): a knowledge-base question gets a grounded answer. An uncovered question gets "don't know" + contact. "Where's my order" returns the seeded order. Asking for another seeded number's order returns not-found.

## Group E — Real business tools (blocked)

- [ ] RAG-T15 Resolve RAG-FR-08 clarification. Add a requirement per tool to `spec.md` and design notes to `design.md`.
  - Refs: RAG-FR-08
  - Done when: the spec status can move to `Ready` for Group E.
- [ ] RAG-T16+ One task pair (tests → implementation) per real tool, following the RAG-T10 pattern.

## Exit criteria

- Groups A–D done and CI green. Group E done, or explicitly deferred with a dated note in `spec.md`.
