# Qdrant + BGE production stack

PostgreSQL remains the canonical application database. Qdrant is a rebuildable
retrieval index containing normalized payload fields plus BGE-M3 `dense` and
learned `sparse` named vectors.

## Current records and historical reports

Normal promise questions, counts, and meter or collection overviews use only
`document_type=promise` records synchronized from the source database. Reports
are excluded from their retrieval filters, prompts, citations, and cached answers.
Missing current records do not cause a fallback to report evidence.

Explicit report requests, such as "summarize the Batometer report", use
`document_type=meter_report`. Their ratings and totals describe the historical
report, not the current database. Direct report summary actions remain available.
Methodology questions use the separate `knowledge` document type.

These are separate evidence scopes within the existing index; no indexed reports
need to be deleted or reindexed. Current-record freshness still depends on source
sync. Configure `SOURCE_SYNC_ENABLED=true` with the source database connection
settings to keep the PostgreSQL and Qdrant copies refreshed. This does not query
the source database on every chat request.

## Runtime flow

1. The BGE-M3 embedding service returns a 1024-dimensional dense vector and a
   learned sparse vector from `POST /embed`.
2. Qdrant prefilters both retrieval legs by normalized metadata, retrieves 40
   candidates per leg, and fuses them with RRF (`k=60`).
3. The separately served `BAAI/bge-reranker-v2-m3` model scores every fused
   candidate through `POST /rerank`.
4. Candidates below the calibrated evidence threshold are discarded. The top
   eight distinct parent documents are packed into the prompt.
5. Gemini Flash (or an OpenAI mini-class model) answers with application-owned
   citations. Conflicting evidence and complex synthesis select the configured
   stronger model. Insufficient evidence is rejected before generation.

The embedding service contract is:

```json
{
  "model": "BAAI/bge-m3",
  "inputs": ["query or document"],
  "return_dense": true,
  "return_sparse": true
}
```

It must return an object containing `dense` and either a sparse
`{indices, values}` object or a token-id-to-weight mapping. Ollama's dense-only
embedding endpoint is not sufficient for this mode.

The reranker follows the common `/rerank` contract: `query`, `documents`,
`model`, and `top_n`, with every result containing `index` and
`relevance_score` (or `score`) calibrated to `[0, 1]`.

Run `promisetracker-index --ensure-schema` to read the bundled local artifact,
create the Qdrant collection if necessary, and upsert both vector
representations. Use `--from-source` only when live database extraction is
intentionally enabled. Set `RETRIEVAL_BACKEND=pgvector` for the retained
rollback path.

Thresholds must be calibrated against the adjudicated evaluation corpus before
production rollout. RRF scores are ranks, not probabilities, and are never
used directly by the evidence refusal gate.

Offline preparation now uses the exact BGE-M3 Hugging Face tokenizer by
default. For deterministic offline deployments, mount its `tokenizer.json` and
run:

```sh
python scripts/prepare_promise_tracker_rag.py \
  --tokenizer-file /models/bge-m3/tokenizer.json
```

The compatibility character-chunk API remains available to older callers, but
the production CLI creates 450-token target chunks, bounded by 350–600 tokens
with 60-token overlap. Each chunk records its exact token count and tokenizer
identity in metadata.

After collecting human-adjudicated calibration scores, run
`scripts/calibrate_thresholds.py` as documented in `evals/README.md`, copy the
recommended values into `RERANKER_MIN_SCORE` and `RETRIEVAL_MIN_SCORE`, then
freeze them before held-out evaluation.

## Container deployment

`docker-compose.yml` runs the application with Qdrant and one local model API
that exposes `/embed`, `/rerank`, and `/health`. PostgreSQL remains external and
must be reachable through the `DATABASE_URL` in the application env file.

1. Copy `.env.compose.example` to a deployment-only Compose env file and replace
   `QDRANT_API_KEY` with a high-entropy secret.
2. Put application credentials (database, Gemini/OpenAI, WhatsApp, and Telegram)
   in the file referenced by `APP_ENV_FILE`. Do not put secrets in the Compose
   file or build arguments.
3. Run `docker compose --env-file /secure/path/promisetracker-compose.env up -d
   --build`.
4. Create or refresh the retrieval index with `docker compose --env-file
   /secure/path/promisetracker-compose.env --profile index run --rm indexer`.
5. Verify `docker compose ps`, `GET http://127.0.0.1:8000/live`, and the API's
   readiness endpoint before routing traffic.

Qdrant data and downloaded model weights live in the `qdrant_data` and
`model_cache` named volumes. Back up Qdrant for faster recovery, but continue to
treat it as a rebuildable derivative of PostgreSQL. The published ports bind to
loopback by default; put TLS and authentication at the ingress boundary if the
API is exposed publicly. Qdrant itself is protected by the required API key.

The default model device is CPU for portability. Set `BGE_DEVICE=cuda` only
on a host whose container runtime exposes a compatible NVIDIA GPU; GPU resource
reservation is intentionally an operator-specific override. Initial model
download can exceed the health-check start period on slow links, in which case
increase the `models.healthcheck.start_period` value in a deployment override.
