# PromiseTracker RAG Chatbot — Comprehensive Documentation

> Start with the [unified documentation](../README.md) for the accepted 2026-10-09 target. This historical reference contains conflicting canonical-database and single-process claims; the [reconciliation table](../docs/architecture.md#reconciled-claims-and-open-verification) explains ownership and required supervision.

A single-process FastAPI service that answers **WhatsApp** and **Telegram** questions
about [PromiseTracker.ng](https://promisetracker.ng) political-accountability records
using retrieval-augmented generation (RAG) with grounded, cited answers.

This document covers: what the system does, how to set it up (local and container),
the architecture and data flow, performance/optimization behavior, and how to prompt
the bot effectively.

> **Doc currency note.** The live default stack is **Qdrant + BGE-M3 + BGE-reranker-v2-m3 + Gemini**.
> The authoritative architecture reference is [qdrant-bge-stack.md](qdrant-bge-stack.md).
> The root `README.md` intro, `promisetracker-rag-chatbot-blueprint.md`, and
> `docs/implementation-plan.md` describe an earlier pgvector + DeepSeek design and are
> partly superseded. When they disagree with this file or `qdrant-bge-stack.md`, trust
> the code and this file.

---

## 1. What the system does

- Receives inbound text via **WhatsApp Cloud API** and **Telegram Bot API** (webhook or long-polling).
- Verifies webhook authenticity (Meta `X-Hub-Signature-256`; Telegram secret-token header).
- Interprets the question (LLM-based intent + metadata filter extraction, with a deterministic fallback).
- Retrieves relevant promise records via **hybrid vector search** (dense + learned sparse, fused with RRF).
- **Reranks** candidates with a dedicated cross-encoder and applies an **evidence gate**.
- Generates a grounded, **cited** answer with an LLM, escalating to a stronger model for complex/disputed queries.
- Caches semantically-similar repeat answers, logs each turn for audit, and enforces per-sender rate limits.

The bot answers **only** from indexed PromiseTracker data. If nothing relevant is
retrieved, it abstains rather than inventing facts.

---

## 2. Architecture

### 2.1 Component overview

```
                WhatsApp Cloud API          Telegram Bot API
                        │                          │
                        ▼                          ▼
              POST /webhook/whatsapp     POST /webhook/telegram   (or long-poll CLI)
                        │                          │
                        └──────────┬───────────────┘
                                   ▼
                        FastAPI app (app/main.py)
                    signature/secret verification
                    rate limiting · correlation IDs
                                   │
                     ┌─────────────┴──────────────┐
            durable ingress?                 direct processing
                     │  (inbox in PG)              │ (semaphore-bounded)
                     ▼                              ▼
              promisetracker-worker  ───────►  AnswerPipeline (app/pipeline.py)
                                                    │
    ┌───────────────────────────────────────────────┼────────────────────────────────┐
    ▼                     ▼                ▼          ▼               ▼                 ▼
Query interpreter   Hybrid embed     Qdrant hybrid  Reranker    Evidence gate      LLM (Gemini)
(LLM + fallback)    (BGE-M3 svc)     search (RRF)   (BGE svc)   (min-score)        + citations
                                                                                        │
                                                                                        ▼
                                                                          Answer cache · turn log (PG)
```

### 2.2 The retrieval + generation pipeline (`app/pipeline.py`)

`AnswerPipeline.answer()` executes in this order:

1. **Load conversation history** (only when `CONVERSATION_ENABLED=true` and a store is wired).
2. **Query rewrite** (optional) — rewrite the question using history for better retrieval.
3. **Query interpretation** — `IntelligentQueryInterpreter` (LLM) maps the question to a
   `search_query`, catalog-constrained `filters`, and an `intent`
   (`search`/`list`/`count`/`status`/`updates`/`catalog`). On any LLM/JSON/validation
   error it falls back to the deterministic parser in `app/filters.py`.
4. **Deterministic short-circuits** (no LLM generation):
   - **Catalog questions** ("what states/categories/meters exist?") → list from metadata.
   - **Inventory questions** ("list/how many broken promises in Lagos") → structured list/count.
5. **Retrieval** — embed the expanded query (dense + sparse via BGE-M3), then
   `search_hybrid` against Qdrant (RRF fusion, `k=60`). Falls back to plain dense search
   if hybrid isn't available.
6. **Rerank** — `BgeReranker` scores every candidate through the model service `/rerank`
   and keeps the top `RERANKER_TOP_N` above `RERANKER_MIN_SCORE`. *(Note: the reranker is
   only applied when the pipeline is constructed with a `reranker`; see §6 caveats.)*
7. **Evidence gate** — `EvidencePolicy.assess()` discards records below
   `RETRIEVAL_MIN_SCORE`. If none pass, the bot returns the no-data answer (abstains).
8. **Escalation decision** — `should_escalate()` picks the stronger model when the top
   score is weak, evidence is **disputed** (same promise with conflicting statuses), or the
   query contains complexity markers (compare/trend/how many/across/…).
9. **Evidence packing** — `pack_evidence()` deduplicates by parent document and packs up to
   8 records / 12,000 chars into `<untrusted_evidence handle="E1">` blocks.
10. **Generation** — the LLM answers using the grounded (or contextual) prompt.
11. **Citations + trim** — `render_citations()` maps `[E1]` handles to real source links;
    `trim_for_whatsapp()` bounds length.
12. **Cache + log** — store in the semantic answer cache (when eligible) and log the turn.

### 2.3 Storage model

| Store | Role | Notes |
| --- | --- | --- |
| **PostgreSQL** | Canonical DB, audit log, cache, durable inbox/outbox | System of record; must be reachable via `DATABASE_URL`. |
| **Qdrant** | Retrieval index (`dense` + `sparse` named vectors) | Rebuildable derivative of Postgres. Default backend. |
| **pgvector** | Fallback retrieval backend | `RETRIEVAL_BACKEND=pgvector`; documented rollback path. |

Key Postgres tables (schema `rag`): `promise_embeddings`, `answer_cache`,
`conversation_turns`, plus durable-ingress tables (`inbox`, `outbox`, `index_versions`,
`schema_migrations`, `rate_limit_windows`). Migrations live in `migrations/` (`001`–`006`)
and are immutable + checksum-verified by `promisetracker-schema-check`.

### 2.4 The BGE model service (`services/bge_model_service/`)

A separate HTTP service owns the two memory-heavy checkpoints and exposes:

- `POST /embed` → BGE-M3 dense (1024-dim) + learned sparse vectors.
- `POST /rerank` → BAAI/bge-reranker-v2-m3 relevance scores (`[0,1]`).
- `GET /health` → readiness (`{"ready": true}`).

Splitting this out is the only microservice boundary — everything else is single-process.
Set `BGE_DEVICE=cuda` only on a GPU host; default is CPU for portability.

### 2.5 Tech stack

- FastAPI (Python 3.12+), Uvicorn, asyncpg
- Qdrant (primary) / pgvector (fallback)
- BGE-M3 embeddings + BGE-reranker-v2-m3 (dedicated service)
- LLM via OpenAI-compatible Chat Completions: **Gemini** (default), DeepSeek, or OpenAI
- aiogram 3.x (Telegram), WhatsApp Cloud API
- Ruff + Black, pytest/pytest-asyncio, uv + Hatchling

---

## 3. Setup

### 3.1 Prerequisites

- Python 3.12+
- PostgreSQL 16 with the `pgvector` extension (`postgresql-16-pgvector`)
- Qdrant (default retrieval backend) — via Docker Compose or standalone
- A BGE-M3/reranker service (Docker Compose `models` service), or Ollama for dense-only local dev
- An LLM API key (Gemini by default)

### 3.2 Local development (bare metal)

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env         # then edit .env
uvicorn app.main:app --reload
```

In `APP_ENV=local`, most production validations are skipped, so you can run with minimal
config. Outside local, `Settings.validate_web_runtime()` enforces required secrets, TLS,
and `ACTIVE_INDEX_RETRIEVAL_ENABLED=true`.

### 3.3 Database bootstrap

Install pgvector **on the PostgreSQL server host** (not just the app host):

```bash
sudo apt-get update
sudo apt-get install postgresql-16-pgvector
sudo systemctl restart postgresql
```

Then create the schema either via the migration runner / indexer:

```bash
promisetracker-index --ensure-schema
```

or by running the SQL in `migrations/001_rag_foundation.sql` … `006_*.sql` in order.
The production DB user should be **read-only** on the PromiseTracker source tables; the
app only writes to the `rag` schema. See `LIVE_DB_SETUP.md` for least-privilege grants and
TLS requirements.

Verify the deployed schema before serving traffic:

```bash
promisetracker-schema-check
```

### 3.4 Index the promises

Configure the source-table mapping in `.env` (`PROMISE_SOURCE_TABLE`, `PROMISE_*_COLUMN`),
or set `PROMISE_EXTRACTION_SQL` for a fully custom query returning the documented aliases
(`promise_id`, `title`, `description`, `status`, `category`, `region`, `politician`,
`source_url`, `date_text`, `metadata`). Then:

```bash
promisetracker-index --ensure-schema --limit 500
# or load precomputed embeddings from the offline pipeline:
promisetracker-index --ensure-schema --from-embedded data/.../embedded_records.json
```

With `RETRIEVAL_BACKEND=qdrant`, indexing creates the collection and upserts both dense and
sparse vectors. Qdrant indexing **requires** the hybrid BGE-M3 embeddings (Ollama's
dense-only endpoint is not sufficient for hybrid mode).

### 3.5 Messaging channels

**WhatsApp** — configure `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_ACCESS_TOKEN`,
`WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_APP_SECRET`. Verification handshake:

```
GET  /webhook/whatsapp?hub.mode=subscribe&hub.verify_token=...&hub.challenge=...
POST /webhook/whatsapp
```

**Telegram** — set `TELEGRAM_BOT_TOKEN`. For local dev without a public HTTPS URL, prefer
long-polling:

```bash
promisetracker-telegram-poll --drop-pending-updates   # deletes webhook by default
promisetracker-telegram-poll --keep-webhook            # only if webhook already disabled
```

For production webhooks:

```bash
promisetracker-telegram-webhook --url https://YOUR_PUBLIC_URL/webhook/telegram --drop-pending-updates
promisetracker-telegram-webhook --info
```

`TELEGRAM_WEBHOOK_SECRET_TOKEN` (1–256 chars, letters/digits/`_`/`-`; **not** the bot token)
is checked via the `X-Telegram-Bot-Api-Secret-Token` header. The app replies immediately and
generates the answer in a bounded background task.

### 3.6 Container deployment (Docker Compose)

`docker-compose.yml` runs **api + qdrant + models**; PostgreSQL stays external.

```bash
cp .env.compose.example /secure/path/promisetracker-compose.env   # set QDRANT_API_KEY (high-entropy)
# put app secrets (DB, Gemini, WhatsApp, Telegram) in the APP_ENV_FILE

docker compose --env-file /secure/path/promisetracker-compose.env up -d --build
# build / refresh the index:
docker compose --env-file /secure/path/promisetracker-compose.env --profile index run --rm indexer
docker compose ps
curl http://127.0.0.1:8000/live
```

Notes:
- Published ports bind to loopback by default; terminate TLS/auth at the ingress boundary.
- Qdrant is protected by the required API key. Data lives in `qdrant_data`; model weights in
  `model_cache` (both named volumes). Treat Qdrant as rebuildable from Postgres.
- First model download can exceed the health-check `start_period` on slow links — raise it
  in a deployment override.

### 3.7 Durable ingress worker

When `DURABLE_INGRESS=true` (default), inbound messages land in the Postgres `inbox`, and a
worker claims and processes them (survives restarts). The same worker runs periodic
maintenance (retention cleanup, cache/rate-limit sweeps):

```bash
promisetracker-worker            # add --maintenance-seconds N to tune sweep cadence (default 300)
```

---

## 4. Configuration reference (selected)

Full list in [.env.example](../.env.example). Defaults are from `app/settings.py`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `APP_ENV` | `local` | `local` relaxes production validations. |
| `DATABASE_URL` | — | Postgres DSN (required outside local). |
| `DATABASE_SSL_MODE` | `disable` | Must be `require` outside local. |
| `LLM_PROVIDER` | `gemini` | `gemini` / `openai` / `deepseek` / custom factory. |
| `GEMINI_DEFAULT_MODEL` | `gemini-3-flash-preview` | Primary answer model. |
| `GEMINI_ESCALATION_MODEL` | `gemini-2.5-pro` | Used for complex/disputed queries. |
| `EMBEDDING_PROVIDER` | `ollama` | `ollama` (local) or `openai`. |
| `BGE_M3_BASE_URL` | `http://localhost:8080` | Hybrid embed service. |
| `RETRIEVAL_BACKEND` | `qdrant` | `qdrant` (default) or `pgvector` (fallback). |
| `RETRIEVAL_TOP_K` | `40` | Candidates fetched per retrieval leg. |
| `RETRIEVAL_MIN_SCORE` | `0.35` | Evidence gate threshold. |
| `HYBRID_RETRIEVAL_ENABLED` | `true` | Dense + sparse RRF fusion. |
| `RERANKER_ENABLED` | `true` | Cross-encoder rerank. |
| `RERANKER_TOP_N` | `8` | Records kept after rerank. |
| `RERANKER_MIN_SCORE` | `0.35` | Rerank cutoff (calibrate before prod). |
| `INTELLIGENT_QUERY_ENABLED` | `true` | LLM query interpretation. |
| `CONVERSATION_ENABLED` | `false` | Multi-turn history (needs encryption key). |
| `ANSWER_CACHE_ENABLED` | `true` | Semantic answer cache. |
| `ANSWER_CACHE_MIN_SCORE` | `0.94` | Cosine similarity for cache hits. |
| `RATE_LIMIT_PER_MINUTE` | `20` | Per-sender limit. |
| `MESSAGE_PROCESSING_CONCURRENCY` | `4` | Concurrent RAG turns across channels. |
| `CORPUS_VERSION` / `INDEX_VERSION` / `PROMPT_VERSION` | `corpus-v1` / `index-v1` / `prompt-v3` | Cache-scope versioning. |

**Thresholds must be calibrated** against the adjudicated eval corpus before production. RRF
scores are ranks, not probabilities, and are never used directly by the evidence gate. Run
`scripts/calibrate_thresholds.py` (see `evals/README.md`), copy recommendations into
`RERANKER_MIN_SCORE` / `RETRIEVAL_MIN_SCORE`, and freeze before held-out evaluation.

---

## 5. Performance and optimization

The design optimizes for **low added latency**, **bounded resource use**, and **retrieval quality**.

**Latency reduction**
- **Semantic answer cache** (`rag.answer_cache`): repeat/paraphrased questions reuse a prior
  answer when cosine similarity ≥ `ANSWER_CACHE_MIN_SCORE` (0.94), skipping retrieval + LLM.
  Cache validity is bound to a **scope key** (tenant/authorization/corpus/index/prompt/model/
  embedding versions), so bumping `CORPUS_VERSION`/`INDEX_VERSION`/`PROMPT_VERSION` cleanly
  invalidates stale answers. TTL: `ANSWER_CACHE_TTL_SECONDS` (3600). Cache is skipped when
  conversation context is active (answers are then context-dependent).
- **Deterministic short-circuits**: catalog and inventory questions are answered directly from
  metadata with **no LLM call** at all.
- **Model tiering**: a cheap default model (Gemini Flash) handles most turns; only complex or
  disputed queries escalate to the stronger, slower model.
- **Async replies**: Telegram/WhatsApp handlers acknowledge fast and generate in the background.

**Throughput / resource control**
- **Connection pooling**: one `asyncpg` pool (`DATABASE_MIN_SIZE`/`MAX_SIZE`).
- **Shared `httpx.AsyncClient`**: a single client for the app lifetime across all outbound HTTP.
- **Semaphore-bounded RAG**: `MESSAGE_PROCESSING_CONCURRENCY` caps concurrent expensive turns so
  one channel cannot starve the other.
- **Durable ingress**: inbox/outbox in Postgres decouple webhook receipt from processing and
  survive restarts; the worker leases, dedups, and retries.
- **Rate limiting**: per-sender `RATE_LIMIT_PER_MINUTE`; in-memory or Postgres-backed backend.

**Retrieval quality**
- **Hybrid search**: dense (semantic) + learned sparse (lexical) fused with **RRF (k=60)** catches
  both meaning-level and keyword-level matches.
- **Cross-encoder rerank**: BGE-reranker-v2-m3 re-scores fused candidates for precision before
  the top-N are packed.
- **Evidence gate + abstention**: low-scoring evidence is dropped; if nothing qualifies the bot
  says so instead of hallucinating.
- **Deduplication by parent document** during packing avoids near-duplicate chunks crowding the prompt.
- **Chunking**: the offline pipeline targets ~450-token chunks (bounded 350–600, 60-token overlap)
  using the exact BGE-M3 tokenizer, recording token counts + tokenizer identity in metadata.

**Cost control**
- Cache hits and deterministic paths avoid LLM spend entirely.
- Flash-tier default keeps per-answer cost low; escalation is selective.
- `max_tokens` bounds on interpreter and reranker LLM calls.

**Tuning levers** (measure, then adjust): `RETRIEVAL_TOP_K` (recall vs. latency),
`RERANKER_TOP_N` (context size vs. focus), `RETRIEVAL_MIN_SCORE`/`RERANKER_MIN_SCORE`
(precision vs. abstention rate), `ANSWER_CACHE_MIN_SCORE` (hit rate vs. staleness risk),
`MESSAGE_PROCESSING_CONCURRENCY` (throughput vs. DB/LLM load), `BGE_DEVICE=cuda` (embed/rerank speed).

---

## 6. Known caveats (as of the 2026-07 review)

These are real gaps, not intentional deferrals — verify against current code before relying on them:

- **Reranker may not be wired on the default path.** `main.py` can build the retriever/pipeline
  without a `reranker` argument, in which case step 6 (rerank) is skipped and the evidence gate
  runs on raw hybrid scores. Confirm the pipeline is constructed with `BgeReranker` before
  assuming rerank is active.
- **`retrieval_expansion_factor` / `retrieval_candidate_multiplier` are ignored on the Qdrant path**
  (`hybrid_search` uses `candidate_limit = limit`), so "fetch top_k×N for reranking" does not engage
  by default.
- **`CrossEncoderReranker`** (the older DeepSeek-LLM pointwise reranker in `reranker.py`) is **dead
  code** — only `BgeReranker` is used in production.
- **Durable worker JSONB decoding**: `worker_cli.process_inbox` reconstructs `ChatTextMessage` from
  `inbox.payload`; asyncpg returns JSONB as `str` unless a codec is registered — verify a round-trip
  works in your deployment.
- **`rate_limit_windows` table**: ensure `migrations/005_rate_limit_windows.sql` is applied when
  using the Postgres rate limiter.
- **Telegram webhook secret** is not force-required in all code paths — always set
  `TELEGRAM_WEBHOOK_SECRET_TOKEN` in production.

---

## 7. How to prompt the bot

The bot is a **domain assistant for Nigerian political-promise accountability**. It answers
only from indexed PromiseTracker records and always cites sources.

### 7.1 What it understands

The query interpreter handles Nigerian English, abbreviations, and named entities, and detects
intent. You can ask in several shapes:

- **Search / status**: "What's the status of the Lagos-Ibadan rail promise?"
- **Updates / progress**: "Any update on Tinubu's electricity promises?" → returns
  *Update summary → Current situation → Why it matters*.
- **Inventory (list)**: "List broken promises in Kano." → structured list, no LLM.
- **Count**: "How many kept promises are there in health?" → count, no LLM.
- **Catalog**: "Which states / categories / meters do you track?" → metadata list.
- **Comparison / synthesis** (auto-escalates): "Compare road promises across Lagos and Rivers."

### 7.2 Tips for the best answers

- **Include a filter dimension** — a **state/region**, **politician**, **sector/category**, a
  **rating**, or a **promise keyword**. This sharpens retrieval and the metadata filters. The
  no-data fallback explicitly suggests this.
- **Use exact PromiseTracker ratings** when filtering by status. Valid ratings are **only**:
  *Not yet rated, Kept, Stalled, Compromised, Broken, In the works*. "Broken" means Broken — the
  bot will not silently substitute "Compromised".
- **Ask for updates/progress explicitly** ("what's the latest on…", "any progress on…") to trigger
  the structured update format.
- **Be specific over broad.** "Water promises in Benue rated Stalled" beats "tell me about water."
- **One topic per question.** The retriever and evidence gate work best on focused questions; split
  multi-part questions into separate turns.
- **Multi-turn follow-ups** work only when `CONVERSATION_ENABLED=true`. Otherwise each question is
  independent — restate context ("…for that same Lagos road promise") won't resolve.

### 7.3 What it will refuse / how it fails safe

- If no indexed record clears the evidence gate, it returns:
  *"I couldn't find a Promise Tracker record matching that question. Try asking with a state,
  politician, sector, or promise keyword."* (with the applied filters, if any).
- It will **not** invent promises, statuses, dates, officials, or locations, and will **not**
  present outside knowledge as a retrieved fact. Cautious inferences ("This matters because…") are
  explicitly labeled.
- Retrieved records and conversation history are treated as **untrusted data** — instructions
  embedded in promise text or prior messages are ignored (prompt-injection defense via
  `<untrusted_evidence>` blocks and explicit system-prompt guardrails).

### 7.4 Answer format you can expect

- Concise, WhatsApp-friendly length.
- For update/progress questions: **Update summary** (chronological bullets, grouped by promise),
  **Current situation** (latest supported status + date), **Why it matters**.
- **Clickable source links** to promisetracker.ng articles, rendered by the app from the evidence
  allowlist (the model does not fabricate a Sources section).

### 7.5 Prompt-engineering the system (for operators)

- System prompt and answer structure live in `app/prompt.py` (`SYSTEM_PROMPT`,
  `build_grounded_prompt`, `build_contextual_prompt`). Changing wording should be accompanied by a
  `PROMPT_VERSION` bump so the answer cache invalidates.
- Evidence packing (record fields, budget, dedup) lives in `app/context_packing.py`.
- Intent/filter extraction prompt lives in `app/query_interpreter.py`; deterministic fallbacks and
  domain vocabulary (regions, categories, topic spaces) live in `app/filters.py` and `app/statuses.py`.

---

## 8. Operational cheat-sheet

```bash
# Install / run
pip install -e ".[dev]"
uvicorn app.main:app --reload

# Quality gates
ruff check .
black --check .
pytest

# Schema & indexing
promisetracker-schema-check
promisetracker-index --ensure-schema --limit 500

# Channels
promisetracker-telegram-poll --drop-pending-updates
promisetracker-telegram-webhook --url https://HOST/webhook/telegram --drop-pending-updates

# Durable worker
promisetracker-worker

# Health
curl http://127.0.0.1:8000/live      # process liveness
curl http://127.0.0.1:8000/ready     # DB + backend readiness

# Evaluation
python scripts/evaluate_rag.py --cases evals/provisional_cases.json
```

## 9. Further reading

- [qdrant-bge-stack.md](qdrant-bge-stack.md) — authoritative architecture/deploy reference.
- [query-response-logging.md](query-response-logging.md) — turn logging (question/prompt/answer).
- [../LIVE_DB_SETUP.md](../LIVE_DB_SETUP.md) — least-privilege DB grants, TLS, smoke checks.
- [../AGENTS.md](../AGENTS.md) — contributor conventions and architectural decisions.
- `../STATE_B_VALIDATION.md` — pending live-infra validation handoffs.
- `../evals/README.md` — evaluation corpus, calibration, and gates.
