# PromiseTracker RAG Chatbot: Fully Local Startup Guide

This guide starts the complete Mai Gaskiya application directly on one Linux
host. Docker is not required. PostgreSQL, Qdrant, the BGE model service,
FastAPI, the durable worker, and optional messaging clients all run as local
processes.

For a deployed host, the systemd units in
[`deploy/systemd`](../deploy/systemd/README.md) manage these local processes,
restart failures, collect logs in the journal, and start the core stack at boot.

## 1. Processes you will run

A functional chatbot uses the following local services:

| Service | Default address | Purpose |
| --- | --- | --- |
| PostgreSQL with pgvector | `127.0.0.1:5432` | Durable messages, cache, migrations, and source records |
| Qdrant | `127.0.0.1:6333` | Hybrid vector retrieval |
| BGE model service | `127.0.0.1:8080` | Dense/sparse embeddings and reranking |
| FastAPI application | `127.0.0.1:8000` | Health checks and messaging webhooks |
| Durable worker | Local background process | Processes inbox messages and sends replies |
| Telegram poller | Optional local process | Receives Telegram messages without a public webhook |

The recommended terminal layout is:

1. PostgreSQL service
2. Qdrant
3. BGE model service
4. FastAPI
5. Durable worker
6. Telegram polling, if enabled

PostgreSQL is the canonical database. Qdrant is a rebuildable search index.

## 2. Prerequisites

Install:

- Linux on an x86-64 host
- Python 3.12 or newer for the main application
- Python 3.11 or newer for the BGE model service
- PostgreSQL 16 with the pgvector extension
- `uv`, or `pip` and Python virtual environments
- `curl`
- Internet access during the initial Python and Hugging Face model downloads
- A Gemini, OpenAI, or DeepSeek API key
- Enough memory and disk space to load and cache BGE-M3 and the reranker

Telegram and WhatsApp credentials are optional until those channels are used.

On Ubuntu, install the core system packages:

```bash
sudo apt-get update
sudo apt-get install \
  postgresql \
  postgresql-client \
  postgresql-16-pgvector \
  python3.12 \
  python3.12-venv \
  curl
```

Package names vary by Linux distribution and PostgreSQL version.

## 3. Configure local PostgreSQL

Start PostgreSQL:

```bash
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
```

Open a PostgreSQL administration shell:

```bash
sudo -u postgres psql
```

Create a local application role and database. Replace the example password:

```sql
CREATE ROLE rag_user WITH LOGIN PASSWORD 'replace-with-a-strong-password';
CREATE DATABASE promisetracker OWNER rag_user;
\connect promisetracker
CREATE EXTENSION IF NOT EXISTS vector;
\q
```

The `rag_user` role owns this local development database, so it can apply the
application migrations. Production deployments should use separate migration
and least-privilege runtime roles as described in
[`LIVE_DB_SETUP.md`](../LIVE_DB_SETUP.md).

Verify the connection:

```bash
psql \
  "postgresql://rag_user:replace-with-a-strong-password@127.0.0.1:5432/promisetracker" \
  -c "SELECT current_database(), extversion FROM pg_extension WHERE extname = 'vector';"
```

## 4. Install the main application

From the repository root:

```bash
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

With `uv`, use:

```bash
uv sync --extra dev
. .venv/bin/activate
```

Confirm the commands were installed:

```bash
python --version
promisetracker-index --help
promisetracker-schema-check --help
promisetracker-worker --help
```

## 5. Install the local BGE model service

Use a separate environment because PyTorch, FlagEmbedding, and Transformers are
large dependencies that the web application itself does not need:

```bash
uv venv --python python3.11 .venv-bge
uv pip install \
  --python .venv-bge/bin/python \
  -r services/bge_model_service/requirements.txt
```

If `python3.11` is unavailable, use a supported local Python version that can
install the declared PyTorch and FlagEmbedding versions.

Verify the exact imports used during service startup:

```bash
.venv-bge/bin/python -c \
  "from FlagEmbedding import BGEM3FlagModel, FlagReranker; from transformers import AutoTokenizer; print('BGE imports OK')"
```

The first model-service start downloads:

- `BAAI/bge-m3`
- `BAAI/bge-reranker-v2-m3`

Set a persistent Hugging Face cache location:

```bash
mkdir -p .model-cache/huggingface
```

## 6. Create the local environment file

Copy the template:

```bash
cp .env.example .env
```

Do not commit `.env`. Generate independent high-entropy secrets:

```bash
openssl rand -hex 32
```

Use this local baseline:

```dotenv
APP_ENV=local
LOG_LEVEL=INFO

DATABASE_URL=postgresql://rag_user:replace-with-a-strong-password@127.0.0.1:5432/promisetracker
DATABASE_SSL_MODE=disable
DATABASE_MIN_SIZE=1
DATABASE_MAX_SIZE=5

LLM_PROVIDER=openai
GEMINI_API_KEY=replace-with-real-gemini-key
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_DEFAULT_MODEL=gpt-5.6-sol
OPENAI_ESCALATION_MODEL=gpt-5.6-sol

EMBEDDING_MODEL=BAAI/bge-m3
EMBEDDING_DIMENSIONS=1024
BGE_M3_BASE_URL=http://127.0.0.1:8080

RETRIEVAL_BACKEND=qdrant
QDRANT_URL=http://127.0.0.1:6333
QDRANT_API_KEY=replace-with-a-local-qdrant-key
QDRANT_COLLECTION=promises

HYBRID_RETRIEVAL_ENABLED=true
RERANKER_ENABLED=true
RERANKER_BASE_URL=http://127.0.0.1:8080
RERANKER_MODEL=BAAI/bge-reranker-v2-m3
RETRIEVAL_TOP_K=40
RETRIEVAL_MIN_SCORE=0.10
RERANKER_MIN_SCORE=0.10

PRIVACY_HMAC_KEY=replace-with-an-independent-secret
PRIVACY_KEY_VERSION=privacy-v1
DURABLE_INGRESS=true
RATE_LIMIT_BACKEND=postgres
MESSAGE_PROCESSING_CONCURRENCY=4

ANSWER_CACHE_ENABLED=true
ANSWER_CACHE_MIN_SCORE=0.94
PROMPT_VERSION=mai-gaskiya-v2-taxonomy
```

Do not disable the reranker when using Qdrant. Qdrant's reciprocal-rank-fusion
scores are not calibrated confidence values and cannot safely drive the evidence
gate.

For OpenAI instead of Gemini:

```dotenv
LLM_PROVIDER=openai
OPENAI_API_KEY=replace-with-real-key
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_DEFAULT_MODEL=gpt-5.6-sol
OPENAI_ESCALATION_MODEL=gpt-5.6-sol
```

For DeepSeek:

```dotenv
LLM_PROVIDER=deepseek
DEEPSEEK_API_KEY=replace-with-real-key
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_DEFAULT_MODEL=deepseek-chat
DEEPSEEK_ESCALATION_MODEL=deepseek-reasoner
```

## 7. Start local Qdrant

This repository contains the local Qdrant binary at `.local/bin/qdrant`.

Create its storage directory:

```bash
mkdir -p .local/qdrant/storage
```

In terminal 1, start Qdrant with the same API key configured in `.env`:

```bash
QDRANT__STORAGE__STORAGE_PATH=.local/qdrant/storage \
QDRANT__SERVICE__API_KEY=replace-with-a-local-qdrant-key \
.local/bin/qdrant --disable-telemetry
```

Leave it running. Verify it from another terminal:

```bash
curl -sS \
  -H "api-key: replace-with-a-local-qdrant-key" \
  http://127.0.0.1:6333/healthz
```

If the repository binary is unavailable or incompatible with the host, install
the current Qdrant Linux binary and use the same environment variables.

## 8. Start the local BGE service

In terminal 2:

```bash
HF_HOME="$PWD/.model-cache/huggingface" \
BGE_DEVICE=cpu \
BGE_PRELOAD=true \
.venv-bge/bin/python -m uvicorn services.bge_model_service.api:app \
  --host 127.0.0.1 \
  --port 8080 \
  --workers 1
```

Use exactly one Uvicorn worker. Additional workers load duplicate model copies
into memory.

The initial start can take several minutes because both model checkpoints are
downloaded and loaded. Check readiness from another terminal:

```bash
curl -sS http://127.0.0.1:8080/health
```

Expected after loading:

```json
{"status":"ok","ready":true}
```

For a supported NVIDIA setup, use `BGE_DEVICE=cuda`. CPU is slower but requires
no GPU configuration.

## 9. Apply migrations and index PromiseTracker data

Activate the main environment:

```bash
. .venv/bin/activate
```

The normal indexer uses the bundled
`data/promise_tracker_rag/embedded/embedded_records.json` artifact. This local
bootstrap does not require a PromiseTracker source database. Live extraction is
available later with `--from-source` and these `.env` variables:

```dotenv
PROMISE_SOURCE_TABLE=promises
PROMISE_ID_COLUMN=id
PROMISE_TITLE_COLUMN=title
PROMISE_DESCRIPTION_COLUMN=description
PROMISE_STATUS_COLUMN=status
PROMISE_CATEGORY_COLUMN=category
PROMISE_REGION_COLUMN=state
PROMISE_POLITICIAN_COLUMN=politician_name
PROMISE_SOURCE_URL_COLUMN=source_url
PROMISE_CREATED_AT_COLUMN=created_at
PROMISE_UPDATED_AT_COLUMN=updated_at
```

Leave optional mappings blank when a column does not exist. For a joined or
nonstandard schema, set `PROMISE_EXTRACTION_SQL` to a query returning:

- `promise_id`
- `title`
- `description`
- `status`
- `category`
- `region`
- `politician`
- `source_url`
- `date_text`
- `metadata`

Apply all migrations and test the bundled local data with a small index:

```bash
promisetracker-index --ensure-schema --limit 25
```

If that succeeds, build the complete index:

```bash
promisetracker-index --ensure-schema
```

Validate immutable migration checksums and required columns:

```bash
promisetracker-schema-check
```

The indexer must be able to reach PostgreSQL, Qdrant, and the BGE service.

## 10. Start FastAPI

In terminal 3:

```bash
. .venv/bin/activate
uvicorn app.main:app \
  --host 127.0.0.1 \
  --port 8000 \
  --reload
```

Check liveness:

```bash
curl -sS http://127.0.0.1:8000/live
```

Expected:

```json
{"status":"ok"}
```

Check complete readiness:

```bash
curl -sS http://127.0.0.1:8000/ready
```

Expected:

```json
{
  "status": "ready",
  "database": "ok",
  "retrieval": "qdrant",
  "durable_ingress": "ok"
}
```

Interactive API documentation is available at:

```text
http://127.0.0.1:8000/docs
```

`/live` only proves that FastAPI started. `/ready` also checks PostgreSQL,
migrations, the active index, Qdrant, BGE-M3, and the reranker.

## 11. Start the durable worker

With `DURABLE_INGRESS=true`, webhooks write messages to a PostgreSQL inbox. The
FastAPI process does not consume that durable inbox itself.

In terminal 4:

```bash
. .venv/bin/activate
promisetracker-worker
```

Test a single processing cycle with:

```bash
promisetracker-worker --once
```

Keep the worker running whenever Telegram or WhatsApp messages are expected.

## 12. Telegram locally

Create a bot with BotFather and set:

```dotenv
TELEGRAM_BOT_TOKEN=replace-with-telegram-bot-token
```

For a completely local setup, use polling. It does not require a public HTTPS
address.

In terminal 5:

```bash
. .venv/bin/activate
promisetracker-telegram-poll --drop-pending-updates
```

Telegram does not deliver polling updates while a webhook is registered. The
command removes the existing webhook by default.

The complete Telegram path is:

```text
Telegram poller → PostgreSQL inbox → durable worker → RAG pipeline → Telegram
```

If the bot receives messages but does not reply, confirm both the poller and
durable worker are running.

## 13. WhatsApp from a local application

Meta cannot deliver webhooks directly to `127.0.0.1`. The FastAPI process may
remain local, but its WhatsApp endpoint must be exposed through an HTTPS tunnel
or reverse proxy.

Set:

```dotenv
WHATSAPP_VERIFY_TOKEN=replace-with-your-verification-token
WHATSAPP_ACCESS_TOKEN=replace-with-meta-cloud-api-token
WHATSAPP_PHONE_NUMBER_ID=replace-with-phone-number-id
WHATSAPP_APP_SECRET=replace-with-meta-app-secret
```

Configure Meta with:

```text
Callback URL: https://your-public-tunnel.example/webhook/whatsapp
Verify token: the exact WHATSAPP_VERIFY_TOKEN value
```

Test local verification before configuring the public URL:

```bash
curl -i \
  "http://127.0.0.1:8000/webhook/whatsapp?hub.mode=subscribe&hub.verify_token=YOUR_TOKEN&hub.challenge=12345"
```

The response body should be `12345`. Inbound POST requests are also validated
using `X-Hub-Signature-256`, so `WHATSAPP_APP_SECRET` must match the Meta
application that owns the webhook.

## 14. Optional conversation memory

Conversation memory is disabled by default. Enable it with another independent
secret:

```dotenv
CONVERSATION_ENABLED=true
CONVERSATION_REWRITE_ENABLED=true
CONVERSATION_ENCRYPTION_KEY=replace-with-an-independent-high-entropy-secret
CONVERSATION_MAX_TURNS=8
CONVERSATION_TTL_MINUTES=30
```

Restart FastAPI and the worker after changing `.env`.

## 15. Keep the index synchronized

For a live MariaDB PromiseTracker source, configure a read-only source account:

```dotenv
SOURCE_DB_DRIVER=mariadb
SOURCE_DB_SCHEMA_PROFILE=promisetracker
SOURCE_DB_HOST=127.0.0.1
SOURCE_DB_PORT=3306
SOURCE_DB_NAME=promisetracker
SOURCE_DB_USER=rag_sync_reader
SOURCE_DB_PASSWORD=replace-with-password
SOURCE_DB_SSL_ENABLED=false
```

To pull a reviewable source snapshot and prepare it for indexing first:

```bash
.venv/bin/python -m app.source_sync_cli --prepare-only
```

This reads meters, promises, updates, states, trackers, and media in a consistent
read-only transaction. It writes to `data/promise_tracker_rag/source_db`, keeping
the source snapshot separate from report artifacts. The output includes raw table
JSON, joined promise documents, relationships, a source manifest, and validated
BGE-M3 token chunks in `prepared/index_records.json` and `prepared/index_records.jsonl`.
Only published, non-deleted promises and published, non-rejected updates enter
the prepared records. Current promise ratings take priority over update ratings.
The update rating placeholder "Select Option" is omitted while its evidence text
is retained. Update evidence dates and meter chamber/tenure metadata are preserved.

Use `--out PATH` for another destination and `--tokenizer-file PATH` for a local
BGE-M3 tokenizer. Otherwise the tokenizer loads from the Hugging Face cache or
downloads on its first use. `--from-sql PATH` prepares an existing SQL dump through
the same interface. Preparation does not require a target RAG database, embedding
service, or Qdrant connection and does not modify the source database or index.

To embed these prepared records before importing them:

```bash
.venv/bin/python scripts/embed_prepared_rag.py \
  --source data/promise_tracker_rag/source_db/prepared/index_records.json \
  --out data/promise_tracker_rag/source_db/embedded
promisetracker-index --ensure-schema \
  --from-embedded data/promise_tracker_rag/source_db/embedded/embedded_records.json
```

Perform the initial synchronization:

```bash
. .venv/bin/activate
promisetracker-sync --ensure-schema
```

Run continuous synchronization in another terminal:

```bash
promisetracker-sync --watch --interval 300
```

Or schedule one-shot runs:

```bash
promisetracker-sync
```

The sync process is idempotent. A limited run does not infer deletions.

## 16. Daily startup sequence

After the initial installation and indexing, start the application in this
order:

1. PostgreSQL:

   ```bash
   sudo systemctl start postgresql
   ```

2. Qdrant in terminal 1:

   ```bash
   QDRANT__STORAGE__STORAGE_PATH=.local/qdrant/storage \
   QDRANT__SERVICE__API_KEY=replace-with-a-local-qdrant-key \
   .local/bin/qdrant --disable-telemetry
   ```

3. BGE service in terminal 2:

   ```bash
   HF_HOME="$PWD/.model-cache/huggingface" \
   BGE_DEVICE=cpu \
   BGE_PRELOAD=true \
   .venv-bge/bin/python -m uvicorn services.bge_model_service.api:app \
     --host 127.0.0.1 --port 8080 --workers 1
   ```

4. FastAPI in terminal 3:

   ```bash
   . .venv/bin/activate
   uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
   ```

5. Worker in terminal 4:

   ```bash
   . .venv/bin/activate
   promisetracker-worker
   ```

6. Telegram polling in terminal 5, if used:

   ```bash
   . .venv/bin/activate
   promisetracker-telegram-poll --drop-pending-updates
   ```

7. Verify:

   ```bash
   curl -sS http://127.0.0.1:6333/healthz \
     -H "api-key: replace-with-a-local-qdrant-key"
   curl -sS http://127.0.0.1:8080/health
   curl -sS http://127.0.0.1:8000/live
   curl -sS http://127.0.0.1:8000/ready
   ```

## 17. Run quality checks

Activate the main environment:

```bash
. .venv/bin/activate
pytest
ruff check .
black --check .
```

Run the RAG evaluation when all local dependencies and LLM credentials are
available:

```bash
python scripts/evaluate_rag.py --cases evals/provisional_cases.json
```

## 18. Troubleshooting

### `PostgreSQL rate limiting requires DATABASE_URL`

Confirm `.env` exists in the repository root and `DATABASE_URL` is valid:

```bash
psql \
  "postgresql://rag_user:replace-with-a-strong-password@127.0.0.1:5432/promisetracker" \
  -c "SELECT 1;"
```

### `extension "vector" is not available`

Install the pgvector package matching the local PostgreSQL major version,
restart PostgreSQL, and rerun:

```bash
promisetracker-index --ensure-schema
```

### Qdrant does not start

Check whether port 6333 is already in use:

```bash
ss -ltnp | grep 6333
```

Confirm the binary is executable:

```bash
chmod +x .local/bin/qdrant
.local/bin/qdrant --version
```

### Qdrant returns unauthorized

The `QDRANT__SERVICE__API_KEY` used to start Qdrant must exactly match
`QDRANT_API_KEY` in `.env`. Restart FastAPI after changing the key.

### BGE health remains `loading`

Watch the model-service terminal for download or memory errors. Confirm internet
access, free disk space, and that the Hugging Face cache is writable. Keep only
one Uvicorn worker.

### `FlagEmbedding and transformers are required`

First test the imports with the BGE environment's interpreter:

```bash
.venv-bge/bin/python -c \
  "from FlagEmbedding import BGEM3FlagModel, FlagReranker; from transformers import AutoTokenizer; print('BGE imports OK')"
```

If that succeeds, do not launch the service with a bare `uvicorn` command. A
virtual environment copied or moved from another project path can contain an
old absolute interpreter path in its launcher script. Start it with:

```bash
.venv-bge/bin/python -m uvicorn \
  services.bge_model_service.api:app \
  --host 127.0.0.1 --port 8080 --workers 1
```

If the direct import fails, rebuild the disposable BGE environment:

```bash
mv .venv-bge .venv-bge.old
uv venv --python python3.11 .venv-bge
uv pip install \
  --python .venv-bge/bin/python \
  -r services/bge_model_service/requirements.txt
```

After the rebuilt service starts successfully, the old environment can be
removed.

### `unsupported embedding model`

Use:

```dotenv
EMBEDDING_MODEL=BAAI/bge-m3
RERANKER_MODEL=BAAI/bge-reranker-v2-m3
```

Restart FastAPI, the worker, and the model service after correcting model names.

### `/live` works but `/ready` returns 503

Check each dependency individually:

```bash
psql \
  "postgresql://rag_user:replace-with-a-strong-password@127.0.0.1:5432/promisetracker" \
  -c "SELECT 1;"
curl -sS http://127.0.0.1:6333/readyz \
  -H "api-key: replace-with-a-local-qdrant-key"
curl -sS http://127.0.0.1:8080/health
promisetracker-schema-check
```

Also confirm that indexing completed and created an active index version.

### Indexing reports zero records

Verify `PROMISE_SOURCE_TABLE` and all `PROMISE_*_COLUMN` mappings. Use
`PROMISE_EXTRACTION_SQL` for a joined or nonstandard source schema.

### Messages are accepted but no reply is sent

Confirm the durable worker is running. Then inspect the worker terminal,
channel credentials, PostgreSQL inbox/outbox state, and outbound network access.

### Telegram polling receives nothing

Remove the registered webhook:

```bash
promisetracker-telegram-poll --drop-pending-updates
```

### Port already in use

Identify the process:

```bash
ss -ltnp | grep -E ':(5432|6333|8000|8080)\b'
```

Stop the old process or intentionally configure a different port and update
`.env` to match.

## 19. Stop the local application

Stop FastAPI, the worker, the Telegram poller, Qdrant, and the BGE service with
`Ctrl+C` in their terminals.

PostgreSQL may remain running. To stop it:

```bash
sudo systemctl stop postgresql
```

Do not delete `.local/qdrant/storage` unless you intend to rebuild the complete
Qdrant index. Do not delete `.model-cache/huggingface` unless you intend to
download the model weights again.
