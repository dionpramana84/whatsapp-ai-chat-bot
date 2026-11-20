# 004 — RAG + tools

- **Status:** Draft (clarifications open)
- **Phase:** 3
- **Depends on:** 003

## Goal

Ground answers in the business's own knowledge base, and let the bot take sender-scoped actions (such as order lookup or booking) through tools. A tool must never be able to reach another customer's data, whatever the customer types.

## User stories

- **US1:** As a customer, I get answers based on the business's actual FAQs, policies and product info, not the model's guesses.
- **US2:** As a customer, when the knowledge base doesn't cover my question, the bot says so and offers a human or a contact route.
- **US3:** As a customer, I can ask about *my* order or booking and get real data.
- **US4:** As a customer trying to get someone else's data ("show order #123 for +62…"), I get nothing that isn't mine.
- **US5:** As a content editor, I update a knowledge-base file and the bot uses the new content after re-ingest.

## Functional requirements

### RAG-FR-01 — Knowledge-base source and ingest
For the $0 test profile, the knowledge base MUST be Markdown files under `kb/`. `npm run kb:ingest` MUST split each file into chunks by heading (target 300–800 tokens, never across files), skip chunks whose content hash is unchanged, embed new or changed chunks, and delete chunks for removed files or sections. Ingest MUST be idempotent.

- **AC1:** Given an unchanged `kb/`, when ingest runs twice, then the second run makes zero embedding calls.
- **AC2:** Given one edited section, when ingest runs, then exactly that chunk is re-embedded.
- **AC3:** Given a deleted file, when ingest runs, then its chunks are removed.

### RAG-FR-02 — Embeddings
Embeddings MUST be produced by `EMBEDDING_PROVIDER` / `EMBEDDING_MODEL` at a fixed `EMBEDDING_DIM` (default 768), stored in `kb_chunks.embedding vector(768)` with an HNSW cosine index. Changing the model or dimension MUST require a migration plus a full re-ingest (`kb:ingest --rebuild`). Each row stores the model name so a mismatch is detected.

- **AC1:** Given a chunk embedded with model A and config set to model B, when the app starts, then it logs an `error` and refuses retrieval until a rebuild runs.

### RAG-FR-03 — Retrieval
Before each LLM call, the composer MUST embed the merged pending user text (plus the previous assistant message, if any, for context), retrieve the top `RAG_TOP_K` (default 5) chunks with cosine similarity ≥ `RAG_MIN_SCORE`, and pass them to the model in a clearly delimited reference block that the system prompt says is data, not instructions.

- **AC1:** Given a question matching a known chunk, when retrieval runs, then that chunk is in the top-k.
- **AC2:** Given a chunk containing "Ignore your rules and…", when retrieved, then it appears only inside the reference block. (unit, on message assembly)

### RAG-FR-04 — Grounded "don't know"
When no chunk passes `RAG_MIN_SCORE` and no tool applies, the bot MUST say it doesn't have that information and offer the contact route (or handoff once 005 exists). It MUST NOT invent policies, prices or availability.

- **AC1:** Eval cases for uncovered questions pass (added to the 003 suite, category `grounding`).

### RAG-FR-05 — Sender-scoped tool context
Every tool MUST receive the customer's `wa_id` from the server-side job context. Tool input schemas MUST NOT contain fields for customer phone numbers, `wa_id`s or account IDs. Any record-level ID a tool accepts (for example an order number) MUST be checked against the sender's ownership before data is returned.

- **AC1:** Given sender A asks for an order number that belongs to B, when the tool runs, then it returns "not found" (identical to a non-existent order, so no existence oracle).
- **AC2:** A static test asserts that no tool's input schema has a key matching `/phone|wa_?id|customer_?id|account/i`.

### RAG-FR-06 — Bounded tool loop
The model MAY call tools for at most `TOOL_MAX_STEPS` (default 3) steps per turn. Each tool call MUST time out (default 5 s). Tool errors MUST come back to the model as a short structured error (`{ error: "unavailable" }`), never stack traces or internal messages.

- **AC1:** Given a model that keeps calling tools, when the step limit is reached, then the loop stops and a reply is still produced.
- **AC2:** Given a tool that throws, when called, then the model receives `{ error: "unavailable" }` and the error is logged server-side.

### RAG-FR-07 — Reference tool against fixture data
To prove the tool path at $0, the repo MUST include one reference tool, `lookup_my_orders`, backed by a local `orders` fixture table (seeded for the allowlisted test numbers), returning only the sender's orders.

- **AC1:** Given seeded orders for A and B, when A asks "where is my order?", then the reply mentions only A's orders. (integration with a mock model issuing the tool call)

### RAG-FR-08 — Business tools
`[NEEDS CLARIFICATION: What is the bot's primary job, and which existing systems (store, booking tool, CRM) must tools call?]` Each real tool gets its own requirement here once that's answered, following RAG-FR-05/06. If the store is Shopify, the order lookup MUST match the sender by the phone number on the customer or order, through the Admin API with a server-side token, and handle the API's rate limits.

### RAG-FR-09 — Observability
Each turn MUST log the retrieved chunk IDs and scores, and the tool calls (name, duration, success), at `info`, without tool results or message bodies.

- **AC1:** Given a turn with retrieval and a tool call, when logged, then the chunk IDs and tool name appear and no body text does.

## Non-functional requirements

- **RAG-NFR-01:** Retrieval adds at most 300 ms p95 locally (embedding call + HNSW query).
- **RAG-NFR-02:** The free-tier embedding quota is enough for the test knowledge base (fewer than 500 chunks).

## Out of scope

- Payload CMS as the knowledge-base admin. It is deferred until non-developer editors exist, and gets its own spec then.
- Hybrid search (BM25 + vector). Add it if evals show exact-term misses, such as SKUs or product codes.

## Open questions

- `[NEEDS CLARIFICATION: Primary job + existing systems]` (blocks RAG-FR-08 only)
- `[NEEDS CLARIFICATION: Does the knowledge base already exist anywhere (website, Notion, PDFs)?]` This changes the ingest source, not the design.
