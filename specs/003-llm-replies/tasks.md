# 003 — LLM replies: Tasks

Legend: `[P]` can run in parallel within its group. Tests are written first.

## Group A — Pure functions

- [ ] LLM-T01 [P] Business profile type + `buildSystemPrompt`: tests, then implementation
  - Refs: LLM-FR-01, LLM-FR-08
  - Files: config/business.ts, prompts/system.md, src/llm/system-prompt.ts, test/llm/system-prompt.test.ts
  - Done when: LLM-FR-01.AC1 and LLM-FR-08.AC1 pass. Unknown placeholders throw at build time, not at runtime.

- [ ] LLM-T02 [P] `buildMessages` (history mapping, same-role merge, pending merge): tests, then implementation
  - Refs: LLM-FR-02, LLM-FR-08
  - Files: src/llm/build-messages.ts, test/llm/build-messages.test.ts
  - Done when: LLM-FR-02.AC3 passes. The mapping table in the design is covered case by case.

- [ ] LLM-T03 [P] `postprocess` (4,096 clamp at a boundary, empty → fallback): tests, then implementation
  - Refs: LLM-FR-04
  - Files: src/llm/postprocess.ts, test/llm/postprocess.test.ts
  - Done when: LLM-FR-04.AC1–AC2 pass, including a 5,000-character string with no sentence boundary.

- [ ] LLM-T04 [P] `isWindowOpen` with a 5-minute margin: tests, then implementation
  - Refs: LLM-FR-05
  - Files: src/whatsapp/window.ts, test/whatsapp/window.test.ts
  - Done when: boundary tests at 23h54m, 23h55m and 23h56m pass. `now` is injected.

## Group B — Config and provider

- [ ] LLM-T05 Extend the config schema: `LLM_PROVIDER`, `LLM_MODEL`, provider keys (conditional), `LLM_TIMEOUT_MS`, `HISTORY_MAX_MESSAGES`, `HISTORY_MAX_DAYS`, `FALLBACK_MESSAGE`, `REENGAGE_TEMPLATE`; `DEBOUNCE_SECONDS` default 3
  - Refs: LLM-FR-03, LLM-FR-06, LLM-FR-07
  - Files: src/config.ts, .env.example, test/config.test.ts
  - Done when: LLM-FR-07.AC1–AC2 pass (zod discriminated union on `LLM_PROVIDER`). The env-example parity test passes.

- [ ] LLM-T06 Provider factory `createModel`
  - Refs: LLM-FR-07
  - Files: src/llm/provider.ts, test/llm/provider.test.ts
  - Depends: LLM-T05
  - Done when: each provider value returns a model instance (no network in the test). The pinned AI SDK version and checked docs URL are noted in the PR.

## Group C — Composer

- [ ] LLM-T07 `composeLlm` tests with a mock model (success + usage, timeout×2 → fallback, 429 → retry → success, 4xx → fallback, no retry)
  - Refs: LLM-FR-06, LLM-FR-10
  - Files: test/llm/compose-llm.test.ts
  - Depends: LLM-T01, LLM-T02, LLM-T03, LLM-T06
  - Done when: the tests exist and fail. Use the AI SDK's mock model test utility if the pinned version has one; otherwise inject a `generate` function.

- [ ] LLM-T08 Implement `composeLlm`
  - Refs: LLM-FR-06, LLM-FR-10
  - Files: src/llm/compose-llm.ts
  - Depends: LLM-T07
  - Done when: LLM-T07 passes.

## Group D — Worker integration

- [ ] LLM-T09 Migration: `messages.llm_usage jsonb`
  - Refs: LLM-FR-10
  - Files: src/db/schema.ts, drizzle/000X_*.sql
  - Done when: the migration applies cleanly on top of 002's schema.

- [ ] LLM-T10 History loader query: tests, then implementation
  - Refs: LLM-FR-02
  - Files: src/db/messages.ts, test/db/history.test.ts
  - Done when: LLM-FR-02.AC1–AC2 pass. The query uses `messages_wa_id_created_idx` (check with `EXPLAIN` once and note it in the PR).

- [ ] LLM-T11 Wire `composeLlm` into the reply worker. Window guard + template fallback in the shared `send()`.
  - Refs: LLM-FR-02, LLM-FR-04, LLM-FR-05, LLM-FR-06, LLM-FR-10
  - Files: src/worker/reply.ts, src/whatsapp/client.ts (`sendTemplate`), test/worker/reply.test.ts
  - Depends: LLM-T04, LLM-T08, LLM-T09, LLM-T10
  - Done when: the 002 worker tests still pass with a mock model. LLM-FR-05.AC1–AC3 and LLM-FR-10.AC1 pass.

- [ ] LLM-T12 Debounce integration test (3 messages 500 ms apart → 1 LLM call)
  - Refs: LLM-FR-03
  - Files: test/e2e/debounce.test.ts
  - Depends: LLM-T11
  - Done when: LLM-FR-03.AC1 passes with `DEBOUNCE_SECONDS=1` to keep the test fast.

- [ ] LLM-T13 [P] Typing indicator / read receipt while generating (best effort)
  - Refs: LLM-FR-11
  - Files: src/whatsapp/client.ts, src/worker/reply.ts
  - Depends: LLM-T11
  - Done when: it's implemented if the Graph version supports it (link the doc in the PR), or skipped with a reason written in this task.

## Group E — Evals

- [ ] LLM-T14 Eval runner (`npm run eval`): YAML loader, assertions, judge, summary JSON
  - Refs: LLM-FR-09
  - Files: evals/run.ts, evals/schema.ts, package.json
  - Depends: LLM-T08
  - Done when: it runs against a mock model in a unit test, and against the real provider by hand.

- [ ] LLM-T15 [P] Write ≥ 25 eval cases (≥ 5 out-of-scope, ≥ 5 injection/extraction, ≥ 3 burst, ≥ 3 language, ≥ 2 overlong pressure, the rest in-scope)
  - Refs: LLM-FR-01, LLM-FR-08, LLM-FR-09
  - Files: evals/cases/*.yaml
  - Done when: the cases are reviewed. Synthetic data only (constitution IX).

- [ ] LLM-T16 Run evals on the test provider, iterate on the prompt, and commit results
  - Refs: LLM-FR-09
  - Files: prompts/system.md, evals/results/*.json
  - Depends: LLM-T14, LLM-T15
  - Done when: LLM-FR-09.AC1 is met.

## Group F — Manual end-to-end

- [ ] LLM-T17 Live test on the test number
  - Refs: US1–US4
  - Depends: all above
  - Done when (record in the PR): an in-scope question gets a sensible answer. "Write me a poem about cats" is declined. 3 rapid messages get 1 reply. Setting an invalid API key temporarily produces the fallback message. Messages in two languages are mirrored.

## Exit criteria

- Every box ticked. CI green. Eval results committed.
- Spec status set to `Done` in `specs/README.md`.
