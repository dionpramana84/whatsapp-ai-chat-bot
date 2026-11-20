# Specs — Spec-Driven Development

This project is built spec-first. No code is written without a requirement it satisfies, and no requirement counts as done without a passing test (or a recorded manual check) for each of its acceptance criteria.

## Workflow

```
constitution.md          non-negotiable rules (rarely changes)
      │
      ▼
NNN-feature/spec.md      WHAT and WHY: user stories, requirements, acceptance criteria
      │                  (no implementation detail)
      ▼
NNN-feature/design.md    HOW: modules, data, interfaces, sequence, trade-offs
      │
      ▼
NNN-feature/tasks.md     ordered, traceable, test-first tasks
      │
      ▼
code + tests             each commit references the task ID(s)
```

1. **Specify:** write or refresh `spec.md`. Mark anything undecided `[NEEDS CLARIFICATION: question]`. A spec with open clarifications on in-scope requirements is not ready for design.
2. **Design:** write `design.md`. Check it against the constitution (each design has a "Constitution check" section).
3. **Tasks:** break the design into tasks of half a day or less, each tracing to at least one requirement.
4. **Implement:** pick the next unblocked task, write the failing test, make it pass, tick the box, commit with the task ID.
5. **Close:** when every task is ticked and every acceptance criterion is verified, set the spec status to `Done`.

If implementation shows the spec is wrong, **stop and change the spec first**, then design, then tasks. Don't let code drift from the spec.

## Specs

| ID | Spec | Status | Depends on |
|---|---|---|---|
| 001 | [Foundation](001-foundation/spec.md) | Ready | — |
| 002 | [Echo bot](002-echo-bot/spec.md) | Ready | 001 |
| 003 | [LLM replies](003-llm-replies/spec.md) | Ready | 002 |
| 004 | [RAG + tools](004-rag-tools/spec.md) | Draft (clarifications open) | 003 |
| 005 | [Handoff + media](005-handoff-media/spec.md) | Draft (clarifications open) | 003 |
| 006 | [Production + go-live](006-production/spec.md) | Draft (clarifications open) | 004, 005 |

Status values: `Draft` → `Ready` (no open clarifications on in-scope requirements) → `In progress` → `Done`.

## ID conventions

| Spec | Prefix | Requirement | Task |
|---|---|---|---|
| 001-foundation | `FND` | `FND-FR-01` | `FND-T01` |
| 002-echo-bot | `ECH` | `ECH-FR-01` | `ECH-T01` |
| 003-llm-replies | `LLM` | `LLM-FR-01` | `LLM-T01` |
| 004-rag-tools | `RAG` | `RAG-FR-01` | `RAG-T01` |
| 005-handoff-media | `HND` | `HND-FR-01` | `HND-T01` |
| 006-production | `PRD` | `PRD-FR-01` | `PRD-T01` |

- `FR` = functional requirement, `NFR` = non-functional requirement, `AC` = acceptance criterion (`ECH-FR-02.AC1`).
- Requirement keywords follow RFC 2119: **MUST**, **SHOULD**, **MAY**.
- Acceptance criteria use Given / When / Then and map to one automated test unless tagged `(manual)`.

## Task format

```
- [ ] ECH-T07 [P] Short imperative title
  - Refs: ECH-FR-04, ECH-FR-05
  - Files: src/webhook/handler.ts, test/webhook/handler.test.ts
  - Depends: ECH-T03
  - Done when: AC tests for ECH-FR-04 pass; …
```

- `[P]` means the task can run in parallel with other `[P]` tasks in the same group (different files, no shared dependency).
- Tests come first: the test file in **Files** is written and failing before the implementation.

## Commit convention

```
feat(webhook): verify X-Hub-Signature-256 on raw body

Refs: ECH-T04, ECH-FR-02
```
