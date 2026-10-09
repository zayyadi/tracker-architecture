# PromiseTracker RAG — Data Flows

> Companion to `docs/architecture.md`. Container shows **where**; this page shows **how bits move** in time.

## 1) Query Flow — WhatsApp / Telegram / Web → Answer

End-to-end retrieval-augmented generation. The dashed paths are early exits that skip the LLM.

```mermaid
sequenceDiagram
    actor U as User (WA / TG / Web)
    participant WA as WhatsApp/Telegram API
    participant API as FastAPI app/main.py
    participant INB as rag.inbox<br/>MessageQueue
    participant WRK as InboxWorker
    participant PL as AnswerPipeline<br/>app/pipeline.py
    participant EMB as BGE-M3<br/>models:8080<br/>app/hybrid_embeddings.py
    participant QDR as Qdrant<br/>app/qdrant_store.py
    participant RR as Reranker<br/>models:8080<br/>app/reranker.py
    participant LLM as Gemini<br/>app/llm.py
    participant OUT as rag.outbox
    participant DL as OutboxDispatcher

    U->>WA: natural language question
    WA->>API: POST /webhook/whatsapp or /webhook/telegram<br/>verify signature/secret
    API->>INB: MessageQueue.admit<br/>channel + external_message_id<br/>deduplicate
    INB-->>API: {"status":"accepted"}
    WRK->>INB: lease batch (FOR UPDATE SKIP LOCKED)<br/>poll 0.5s, batch 20, concurrency 20
    INB->>WRK: ChatTextMessage payload

    WRK->>PL: answer(question, retriever, sender_id, channel)
    PL->>PL: greeting/courtesy/tracking/catalog/<br/>inventory early exits<br/>app/pipeline.py:113-277
    alt greeting / courtesy / tracking / inventory
        PL-->>OUT: deterministic answer, no retrieval
    else semantic question
        PL->>PL: has_conversation_context?<br/>load history via PostgresConversationStore<br/>rewrite via ConversationQueryRewriter
        PL->>PL: IntelligentQueryInterpreter<br/>+ infer_filters / semantic_search_filters
        PL->>EMB: embed_hybrid(retrieval_question)<br/>dense 1024 + sparse indices/values
        EMB-->>PL: HybridEmbedding

        PL->>PL: AnswerCache.find(embedding, filters, source_versions)<br/>app/answer_cache.py<br/>cosine ≥0.94, TTL 3600s<br/>skip if history/summary
        alt cache hit and not incomplete
            PL-->>OUT: cached AnswerResult (cached=true)
        else cache miss
            PL->>QDR: search_hybrid(hybrid, filters, top_k=40)<br/>prefetch dense + sparse → RRF k=60<br/>candidate_limit = 40×4×3=480
            QDR-->>PL: PromiseRecord[] with score
            PL->>PL: lexical match guard + filter match
            PL->>RR: rerank(question, records) top_n=8
            RR-->>PL: rescored records (or fallback to RRF order on failure)
            PL->>PL: EvidencePolicy.assess(min_score=0.35)<br/>app/evidence.py
            alt ABSTAIN (no record ≥0.35)
                PL-->>OUT: format_no_data_answer<br/>no LLM call
            else ANSWER
                PL->>PL: pack_evidence ≤8 rec, 12k chars<br/><untrusted_evidence handle="E1">
                PL->>PL: build_grounded_prompt / build_contextual_prompt
                PL->>LLM: generate(model=gemini-3-flash-preview<br/>→ gemini-3.1-pro-preview if escalated,<br/>max_tokens 2400)
                LLM-->>PL: answer with [E1] handles
                PL->>PL: render_citations + trim_for_whatsapp<br/>should_escalate() on complexity
                PL->>PL: AnswerCache.store + CacheScope<br/>corpus/index/prompt/model/embedding
            end
        end
    end

    PL->>PL: log_turn → rag.conversation_turns<br/>HMAC pseudonym + pgp_sym_encrypt<br/>app/main.py:946
    WRK->>OUT: OutboundMessage<br/>idempotency_key=sha256(channel:msg:answer-v1)
    DL->>OUT: lease & send via WhatsAppClient/TelegramClient
    OUT->>WA: send_text / send_answer (with PromiseSummaryAction)
    WA->>U: answer + [E1] → https://promisetracker.ng/...<br/>or abstention copy
```

**Payloads on the wire:**

* **Inbound:** `ChatTextMessage` (`app/models.py:7`) — `channel`, `message_id`, `sender_id`, `text`, `timestamp`.
* **Embedding text → Qdrant:** `Title/Description/Status/Category/Region/Politician/Admin/Date` join (`app/indexing.py:27`, `app/qdrant_store.py:387`).
* **Retrieval filters:** `RetrievalFilters` (`app/models.py:126`) — `region/status/category/meter_name/promise_tracker_name/collection_name/collection_member[_type]/document_type/topic_terms` → `qdrant_filter` casefolded match (`app/qdrant_store.py:22`).
* **Outbound answer:** `AnswerResult` (`app/models.py:145`) — `answer` (already citation-rendered), `sources: PromiseRecord[]`, `model`, `escalated`, `cached`, `retrieval_score`, `actions: PromiseSummaryAction[]` (`app/models.py:25`) rendered as WhatsApp interactive buttons / Telegram inline keyboards (`app/interactions.py:build_summary_actions`).

**Abstention copy (what partners audit):**

> Deterministic via `app/prompt.py:format_no_data_answer` + `format_empty_inventory_answer`. Example: *"I don't have verified information on this in our tracked promises. Explore all promises at https://promisetracker.ng."* No `[E1]` links appear in abstention. The branch is taken **without an LLM call** when `EvidenceDecision.ABSTAIN` (`app/evidence.py:7`).

---

## 2) Indexing Flow — Source DB → Promise Index

One-way, read-only. The RAG service owns the copy; the source is never mutated.

```mermaid
sequenceDiagram
    participant SRC as Source DB<br/>MariaDB/Postgres<br/>PROMISE_SOURCE_TABLE
    participant CLI as promisetracker-index<br/>app/cli.py + app/indexing.py
    participant PG as Postgres (RAG)<br/>rag.promise_embeddings<br/>rag.index_versions
    participant MOD as Model Service<br/>BGE-M3 :8080
    participant QDR as Qdrant<br/>collection promises

    Note over SRC,CLI: Current — periodic pull (solid)
    CLI->>CLI: --ensure-schema → apply_migrations<br/>dimensions 1024, pgvector
    CLI->>SRC: SELECT promise_id/title/description/<br/>status/category/region/<br/>politician/source_url/date_text<br/>FROM {promise_source_table}<br/>ORDER BY updated_at DESC<br/>app/indexing.py:41 build_extraction_sql
    SRC-->>CLI: rows
    CLI->>CLI: parse → PromiseRecord[]<br/>app/models.py:104<br/>+ normalize_taxonomy_metadata
    CLI->>MOD: embed_hybrid(text) × batch 32<br/>text = Title..Date join
    MOD-->>CLI: HybridEmbedding {dense, sparse}
    CLI->>QDR: ensure_collection dense 1024 Cosine + sparse<br/>PUT /collections/promises
    CLI->>QDR: upsert points (id=uuid5 promisetracker:{id})<br/>payload = casefolded filters + metadata<br/>app/qdrant_store.py:334
    CLI->>PG: upsert_embedded_records +<br/>rag.promise_embeddings ON CONFLICT promise_id DO UPDATE<br/>app/indexing.py:129 + app/vector_store.py
    CLI->>PG: register_active_qdrant_index → rag.index_versions<br/>status active, config {backend, collection, indexed_records}<br/>app/cli.py:177

    Note over SRC,CLI: Target — push / CDC (dashed, roadmap)
    SRC-.->CLI: webhook / LISTEN/NOTIFY / Debezium event<br/>on INSERT/UPDATE<br/>(not yet default)
    CLI-.->MOD: embed incremental change
    CLI-.->QDR: upsert single point + PG mirror

    Note over CLI,PG: Legacy/offline path
    CLI->>CLI: --from-embedded<br/>data/promise_tracker_rag/embedded/embedded_records.json<br/>+ meter_reports_embedded
    CLI->>QDR: index_records (same hybrid path)
    CLI->>PG: upsert_embedded_records
```

**What's indexed (and therefore citable):**

| Field | Source → `PromiseRecord` → Qdrant payload / PG column |
|---|---|
| `promise_id` | `promise_id_column` (`id`) → `PromiseRecord.promise_id` → `payload.promise_id`, PK `rag.promise_embeddings.promise_id`, Qdrant `id = uuid5(...)` |
| `title` | `promise_title_column` → `title` → `payload.title` |
| `description` | `promise_description_column` → `description` → part of embedding text + `<untrusted_evidence>` |
| `status` | `promise_status_column` → `status` (verbatim `Broken` etc.) → casefolded filter `status`, rendered as `Status:` in evidence |
| `category` | `promise_category_column` → `category` |
| `region` | `promise_region_column` (`state`) → `region` |
| `politician` | `promise_politician_column` (`politician_name`) → `politician` |
| `source_url` | `promise_source_url_column` → `source_url` → `Source URL:` in evidence + final clickable link |
| `date_text` | `COALESCE(updated_at, created_at)::date::text` (`app/indexing.py:88`) → `date_text` |
| `metadata` | `normalize_taxonomy_metadata` (`app/taxonomy.py`) → `tracker_name/collection_*/branded_meter_name/meter_kind/document_type/taxonomy_version/corpus_version/index_version/content_checksum/parent_id/chunk_id` → filterable Qdrant payload + `rag.promise_embeddings.metadata::jsonb` |

The **embedded text template** is identical in both Postgres and Qdrant paths — guarantees retrieval parity.

**Operations:**

* Manual: `promisetracker-index --ensure-schema` (once), then `promisetracker-index --ensure-schema --limit 500` or `--from-embedded data/.../embedded_records.json` (`AGENTS.md`).
* Compose one-shot: `docker compose --profile index run --rm indexer` (`docker-compose.yml:82`, `restart: no`).
* Continuous (config-driven): `SOURCE_SYNC_INTERVAL_SECONDS` (`app/settings.py:130`, default 300) is the contract for the future poll loop. Until wired to a scheduler, document the cron you actually run (e.g., systemd timer / K8s CronJob invoking the indexer).
* Health gate: `/ready` requires `rag.schema_migrations` current, `rag.inbox` exists, and one `rag.index_versions` row with `status='active'` (`app/main.py:370`).

---

## 3) Supporting Flows (appendix detail — not in main diagram)

```mermaid
flowchart TB
    subgraph Cache
        A[Question embedding] --> B{AnswerCache.find<br/>cosine ≥ 0.94<br/>scope = corpus/index/prompt/model/embedding}
        B -- hit --> C[return cached AnswerResult<br/>skip retrieval+LLM]
        B -- miss --> D[full pipeline]
        D --> E[AnswerCache.store]
    end

    subgraph Privacy
        F[sender_id / message_id] --> G[HMAC-SHA256 pseudonymize<br/>privacy-v1<br/>app/privacy.py]
        G --> H[storage_sender_id → rag.conversation_turns<br/>+ pgp_sym_encrypt question/answer/prompt]
    end

    subgraph Observability
        I[x-request-id header] --> J[correlation_middleware<br/>app/observability.py]
        J --> K[structured_event logging<br/>+ log_turn source_ids/retrieval_score/latency_ms]
    end
```

* Cache invalidation is **scope-keyed**, not TTL-only — bumping `corpus_version / index_version / prompt_version` (`app/settings.py:107`) implicitly evicts old answers.
* Conversation memory (`conversation_enabled`) is off by default; when enabled, history is loaded per `channel+sender_id` and rewritten via `ConversationQueryRewriter` before retrieval (`app/pipeline.py:160`).
* Rate limiting and durability are Postgres-backed in production (`rate_limit_backend=postgres`, `durable_ingress=true`); `single_replica_mode` is required for the in-memory limiter (`app/settings.py:284`).

