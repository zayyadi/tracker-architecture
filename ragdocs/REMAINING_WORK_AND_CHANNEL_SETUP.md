# Remaining Work and Three-Path Setup

> The [unified architecture](../docs/architecture.md) and [channel contracts](../docs/channel-contracts.md) define the accepted 2026-10-09 target. This audit remains supporting implementation evidence, not proof of live deployment.

This document describes what is implemented, what remains before a dependable
production launch, and how to run the application through its three supported
user channels:

1. PromiseTracker.ng web chat at `POST /chat/web`
2. WhatsApp Cloud API webhook
3. Telegram, using either webhook or long polling

Telegram webhook delivery and polling are alternative ingress modes and must
not run at the same time. The web channel is a JSON API for the
PromiseTracker.ng frontend, not a standalone browser UI. `/live`, `/health`,
and `/ready` remain operational endpoints.

## Current State

The repository already contains:

- the FastAPI webhook application;
- WhatsApp signature verification and Telegram webhook-secret verification;
- a durable PostgreSQL inbox and outbox;
- a separate durable worker for RAG processing and outbound delivery;
- Qdrant hybrid retrieval with BGE-M3 embeddings and BGE reranking;
- pgvector fallback retrieval;
- Gemini, OpenAI, and DeepSeek LLM clients;
- answer caching, rate limiting, conversation storage, and source citations;
- schema migrations and schema validation;
- source indexing and incremental source synchronization commands;
- systemd units for the API, worker, model service, Qdrant, source sync, and
  Telegram polling;
- tests and deterministic evaluation tooling.

At the time of this audit, Qdrant responded locally on port `6333`, while the
FastAPI API and BGE model service were not running. This is only a snapshot of
the inspected host, not a repository defect.

## Remaining Work

### Required before production traffic

- [ ] Provision a production PostgreSQL database with `pgvector`, TLS, backups,
  and separate migration/runtime roles.
- [ ] Create deployment secrets. At minimum these include the selected LLM key,
  a Qdrant API key, `PRIVACY_HMAC_KEY`, WhatsApp credentials, and Telegram
  credentials if Telegram is enabled.
- [ ] Set `APP_ENV=production`, `DATABASE_SSL_MODE=require`, and
  `ACTIVE_INDEX_RETRIEVAL_ENABLED=true`. Production startup validation rejects
  unsafe or incomplete values.
- [ ] Start Qdrant and the BGE model service, apply all migrations, load the
  initial index, and run `promisetracker-schema-check`.
- [ ] Keep `promisetracker-worker` running whenever webhook or polling traffic
  is accepted. Webhooks enqueue messages; the API alone does not generate and
  deliver durable replies.
- [ ] Put the FastAPI service behind a public HTTPS reverse proxy for webhook
  operation. Expose port `8000` only to the proxy.
- [ ] Configure and verify WhatsApp and Telegram in their provider consoles.
- [ ] Run real end-to-end tests in every enabled ingress mode and confirm that
  every answer contains a clickable `promisetracker.ng` source URL.
- [ ] Configure logs, alerts, database/Qdrant backups, queue-depth monitoring,
  provider error monitoring, and retention cleanup.
- [ ] Run human-adjudicated RAG evaluation against live infrastructure and
  calibrate retrieval/reranker thresholds before making production-quality
  claims. The bundled provisional cases test the evaluator, not answer quality.

### Repository and documentation follow-up

- [ ] Add the worker (and optionally source sync and Telegram polling) to
  `docker-compose.yml` if Compose is expected to be the complete production
  supervisor. The current Compose stack starts Qdrant, models, API, and an
  on-demand indexer only.
- [ ] Reconcile `.env.example` with runtime defaults. For example, the settings
  default and example currently name different Gemini Flash and BGE/Ollama
  model identifiers.
- [ ] Restore or remove the missing `LIVE_DB_SETUP.md` and
  `STATE_B_VALIDATION.md` links referenced by existing documentation.
- [ ] Perform provider-version review before launch. In particular, verify the
  configured Meta Graph API version and LLM model names against the deployment
  accounts.
- [ ] Add automated live smoke tests for Meta, Telegram, the selected LLM,
  Qdrant, BGE embeddings, reranking, and the production database.

## Recommended Production Topology

```text
WhatsApp ───────┐
                ├─> HTTPS reverse proxy ─> FastAPI ─> PostgreSQL inbox
Telegram webhook┘                                      │
                                                       v
Telegram poller ───────────────────────────────> PostgreSQL inbox
                                                       │
                                                       v
                                                durable worker
                                             /         |         \
                                          BGE       Qdrant       LLM
                                                       │
                                                       v
                                                PostgreSQL outbox
                                                       │
                                             WhatsApp or Telegram
```

Use one API process plus one worker initially. Increase worker capacity only
after measuring database, embedding, reranker, and LLM limits.
`MESSAGE_PROCESSING_CONCURRENCY` bounds expensive RAG turns within a process.

## 1. Base Installation

### Host preparation

Install Python 3.12 or newer, PostgreSQL with `pgvector`, and either Docker or
the host dependencies needed by the supplied systemd units.

```bash
cd /path/to/rag
python3 -m venv .venv
. .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env
```

Generate secrets with a password manager or a cryptographically secure secret
generator. Do not commit `.env`.

### Minimum shared configuration

Use this as a checklist rather than a file to copy verbatim:

```dotenv
APP_ENV=production
LOG_LEVEL=INFO

DATABASE_URL=postgresql://runtime_user:password@db.example:5432/promisetracker
DATABASE_SSL_MODE=require
DATABASE_MIN_SIZE=1
DATABASE_MAX_SIZE=5

LLM_PROVIDER=gemini
GEMINI_API_KEY=replace-me
GEMINI_DEFAULT_MODEL=replace-with-an-enabled-flash-model
GEMINI_ESCALATION_MODEL=replace-with-an-enabled-reasoning-model

RETRIEVAL_BACKEND=qdrant
QDRANT_URL=http://127.0.0.1:6333
QDRANT_API_KEY=replace-with-a-long-random-secret
QDRANT_COLLECTION=promises
BGE_M3_BASE_URL=http://127.0.0.1:8080
RERANKER_BASE_URL=http://127.0.0.1:8080
RERANKER_ENABLED=true
ACTIVE_INDEX_RETRIEVAL_ENABLED=true

PRIVACY_HMAC_KEY=replace-with-a-long-random-secret
ANSWER_CACHE_ENABLED=true
DURABLE_INGRESS=true
RATE_LIMIT_BACKEND=postgres
MESSAGE_PROCESSING_CONCURRENCY=4
```

If conversation memory is required, add:

```dotenv
CONVERSATION_ENABLED=true
CONVERSATION_REWRITE_ENABLED=true
CONVERSATION_ENCRYPTION_KEY=replace-with-a-separate-long-random-secret
```

The production validator requires WhatsApp settings outside local mode even if
only Telegram is being tested. Either provide valid WhatsApp settings or change
the validation policy in `app/settings.py` before operating a Telegram-only
production deployment.

### Start retrieval dependencies

For a Compose-based dependency stack:

```bash
export QDRANT_API_KEY='replace-with-the-deployment-secret'
docker compose up -d qdrant models
docker compose ps
```

The first model startup can take several minutes while weights are downloaded
and loaded. On a CPU-only host, expect materially slower embedding and reranking
than on a supported GPU.

Verify the dependencies:

```bash
curl -fsS -H "api-key: $QDRANT_API_KEY" http://127.0.0.1:6333/readyz
curl -fsS http://127.0.0.1:8080/health
```

### Apply schema and build the active index

Run migrations with a role allowed to create the `rag` schema, extensions,
tables, and indexes:

```bash
promisetracker-index --ensure-schema
promisetracker-schema-check
```

The indexer uses the bundled embedded records by default. For ongoing live
source updates, configure the read-only `SOURCE_DB_*` MariaDB credentials and
run:

```bash
promisetracker-sync --once
```

Run `promisetracker-sync` continuously, or use the supplied systemd sync unit,
only after the one-shot sync succeeds.

### Start the common runtime

In development terminals:

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

```bash
promisetracker-worker
```

In production, prefer the units documented in
[`deploy/systemd/README.md`](../deploy/systemd/README.md):

```bash
sudo systemctl enable --now promisetracker-core.target
```

Check liveness and full dependency readiness:

```bash
curl -fsS http://127.0.0.1:8000/live
curl -fsS http://127.0.0.1:8000/ready
promisetracker-schema-check
```

`/live` only confirms that the API process is alive. Route production traffic
only when `/ready` returns HTTP 200.

## 2. Path A: WhatsApp Cloud API Webhook

Add to `.env`:

```dotenv
WHATSAPP_VERIFY_TOKEN=replace-with-a-private-verification-token
WHATSAPP_ACCESS_TOKEN=replace-with-a-system-user-access-token
WHATSAPP_PHONE_NUMBER_ID=replace-with-the-cloud-api-phone-number-id
WHATSAPP_APP_SECRET=replace-with-the-meta-app-secret
WHATSAPP_GRAPH_BASE_URL=https://graph.facebook.com/replace-with-supported-version
```

In the Meta application:

1. Add the WhatsApp product and connect the sending phone number.
2. Set the callback URL to
   `https://chat.example.org/webhook/whatsapp`.
3. Enter the exact value of `WHATSAPP_VERIFY_TOKEN`.
4. Subscribe the application to the WhatsApp `messages` field.
5. Use a production system-user token with only the permissions required for
   Cloud API messaging.

Smoke-test verification:

```bash
curl -fsS \
  "https://chat.example.org/webhook/whatsapp?hub.mode=subscribe&hub.verify_token=YOUR_TOKEN&hub.challenge=12345"
```

The response must be `12345`. Then send a real WhatsApp text message and verify:

- the webhook returns quickly with HTTP 200;
- an inbox row is admitted once, even if Meta retries;
- the worker creates and delivers one outbox reply;
- the final answer contains clickable PromiseTracker source links;
- invalid `X-Hub-Signature-256` requests are rejected.

## 3. Path B: Telegram Webhook

Add to `.env`:

```dotenv
TELEGRAM_BOT_TOKEN=replace-with-the-BotFather-token
TELEGRAM_WEBHOOK_SECRET_TOKEN=replace_with_a_different_random_secret
TELEGRAM_API_URL=https://api.telegram.org
```

The webhook secret may contain only letters, digits, `_`, and `-`, and must be
1–256 characters. It must not be the bot token.

Make `/webhook/telegram` publicly reachable over HTTPS, then register it:

```bash
promisetracker-telegram-webhook \
  --url https://chat.example.org/webhook/telegram \
  --drop-pending-updates
promisetracker-telegram-webhook --info
```

Send the bot a real text message and verify the inbox, worker, outbox, response,
and source links. Confirm that a request with a missing or incorrect
`X-Telegram-Bot-Api-Secret-Token` is rejected.

Do not start `promisetracker-telegram-poll` while this webhook is registered.

## 4. Path C: Telegram Long Polling

Polling is the preferred local-development path and is also useful when no
public inbound HTTPS endpoint is available. It still requires PostgreSQL, the
retrieval services, the API runtime dependencies, and the durable worker.

Configure `TELEGRAM_BOT_TOKEN`, start the common runtime, then run:

```bash
promisetracker-telegram-poll --drop-pending-updates
```

By default the poller deletes the Telegram webhook because Telegram does not
serve `getUpdates` while a webhook is registered. `--drop-pending-updates`
discards queued messages, so use it deliberately during initial setup rather
than on every normal restart.

For production polling with systemd:

```bash
sudo systemctl enable --now promisetracker-telegram-poll.service
sudo systemctl status promisetracker-telegram-poll.service
```

Do not expose `/webhook/telegram` as an active provider callback in polling
mode.

## 5. Effective Operation

### Startup order

1. PostgreSQL
2. Qdrant
3. BGE embedding/reranker service
4. migrations and active index
5. FastAPI
6. durable worker
7. source sync, if enabled
8. Telegram poller, only in polling mode

### Routine checks

```bash
curl -fsS http://127.0.0.1:8000/ready
promisetracker-schema-check
promisetracker-telegram-webhook --info
```

The Telegram webhook-info command is relevant only in webhook mode. Also monitor:

- inbox rows stuck in `pending`, `processing`, or retry states;
- outbox rows in retry or unknown-delivery states;
- worker/provider exceptions and response latency;
- Qdrant and model-service readiness;
- index version, record count, source URL coverage, and last successful sync;
- PostgreSQL pool saturation, disk use, backup success, and retention cleanup;
- LLM, Meta, and Telegram quota/rate-limit responses.

### Release verification

Before each release:

```bash
ruff check .
black --check .
pytest
python scripts/evaluate_rag.py \
  --cases evals/provisional_cases.json \
  --predictions evals/provisional_predictions.json
```

Also test these domain rules manually:

- only `Not yet rated`, `Kept`, `Stalled`, `Compromised`, `Broken`, and
  `In the works` appear as ratings;
- a request for `Broken` promises never returns `Compromised` promises as if
  they were broken;
- grounded responses include clickable PromiseTracker article links;
- low-evidence questions abstain instead of inventing an answer;
- duplicate provider deliveries do not produce duplicate replies.

## 6. Troubleshooting

### `/ready` returns 503

Check PostgreSQL connectivity, migration version, durable inbox tables, active
index status, Qdrant readiness, and whether both BGE models are loaded. `/live`
can return 200 while these dependencies are unavailable.

### Webhooks return 200 but users get no reply

Confirm that `promisetracker-worker` is running. Then inspect inbox/outbox state
and outbound provider credentials. Durable webhook admission and reply delivery
are separate stages.

### Telegram polling receives nothing

Inspect webhook state:

```bash
promisetracker-telegram-webhook --info
```

Delete the webhook before polling. Avoid discarding pending updates unless that
is intended.

### WhatsApp verification fails

Confirm that the callback uses HTTPS, the route is exactly
`/webhook/whatsapp`, and the Meta verify token exactly matches
`WHATSAPP_VERIFY_TOKEN`.

### Answers contain no clickable sources

Check that indexed records have valid `source_url` values pointing to
PromiseTracker articles. Treat missing source coverage as an indexing/data
quality failure, not a prompt-only problem.

