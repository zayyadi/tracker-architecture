# Query, Prompt, and Response Logging

## Goal

PromiseTracker records each completed chatbot turn in PostgreSQL so operators can
audit questions, reproduce model behavior, evaluate answer quality, and load recent
conversation history. A turn can now contain three distinct text values:

- `question`: the original text received from WhatsApp or Telegram;
- `prompt`: the final grounded prompt sent to the LLM, including packed evidence and
  conversation context when used;
- `answer`: the final user-visible response after citation rendering and WhatsApp
  length trimming.

These values must not be confused. The question represents user intent, the prompt
represents model input, and the answer represents delivered application output.

## Architecture

The existing `rag.conversation_turns` append-only audit table remains the system of
record. Extending it avoids a second logging subsystem, keeps retention behavior
consistent, and allows one database operation to persist all turn data.

```text
channel message
      |
      v
AnswerPipeline.answer
  |-- deterministic answer ----------> AnswerResult(prompt=None)
  |-- semantic cache hit ------------> AnswerResult(prompt=cached prompt)
  `-- retrieval + LLM generation ----> AnswerResult(prompt=final LLM prompt)
                                              |
                                              v
                                  log_turn: one async INSERT
                                              |
                                              v
                                  rag.conversation_turns
```

The pipeline owns prompt construction, so it attaches the exact prompt to
`AnswerResult` at the point where it is known. The transport layer owns message
identity, privacy, and persistence, so `log_turn` writes the result after the answer
has completed. This keeps prompt construction out of the database layer and keeps
channel-specific identifiers out of the RAG pipeline.

## Database design

Migration `006_conversation_prompt_logging.sql` adds two nullable columns:

| Column | Purpose |
| --- | --- |
| `prompt` | Plaintext in local/non-pseudonymized operation, or `[redacted]` when privacy protection is enabled |
| `prompt_ciphertext` | OpenPGP symmetric ciphertext produced by PostgreSQL `pgp_sym_encrypt` when an encryption key is configured |

The pre-existing `question`, `question_ciphertext`, `answer`, and
`answer_ciphertext` columns use the same policy. Prompt columns are nullable because
catalog, inventory, evidence-abstention, and other deterministic paths do not call an
LLM. A cache hit stores the prompt originally used to produce the cached response.

No prompt index is created. Prompts are large, high-cardinality text values and are
not part of the conversation-history lookup path. Existing indexes cover the actual
access patterns:

- `(sender_id, created_at DESC)` for subject history;
- `(channel, created_at DESC)` for channel/time analysis;
- partial `expires_at` index for retention cleanup.

## Runtime implementation

### Generated answers

`AnswerPipeline` already constructs the grounded or contextual prompt immediately
before `LlmClient.generate`. The same immutable string is now returned in
`AnswerResult.prompt`. The value is attached after generation so the logged prompt
corresponds to the model call that produced the answer.

### Cached answers

`AnswerCache` already stores the original prompt with each answer. On a semantic
cache hit, `AnswerPipeline` copies `CachedAnswer.prompt` to `AnswerResult.prompt`.
This makes audit rows reproducible even when the current request did not invoke the
LLM.

### Deterministic answers

Inventory, catalog, and no-evidence responses do not have an LLM prompt and retain
the `AnswerResult` default of `None`. PostgreSQL stores `NULL` in both prompt
columns, accurately distinguishing “no model call” from an empty prompt.

### Persistence

`log_turn` sends all metadata and content in one parameterized `asyncpg.execute`
call. Encryption is performed inside the same `INSERT`; there is no preliminary
query and no follow-up update. Parameters are used for every content value, avoiding
SQL interpolation and preserving prepared-statement reuse.

If a deployment has not yet applied migration 006, the existing legacy-schema
fallback still records the original question and answer using the old column set.
The warning identifies that prompt capture is unavailable until migration rollout.

## Performance characteristics

The change adds column payload to the existing turn insert but does not add a
database round trip. Its expected cost is therefore:

- one pooled connection already held by message processing;
- one asynchronous insert per completed turn;
- optional in-database symmetric encryption for three text values;
- no prompt index maintenance;
- no prompt read during normal conversation-history loading.

Prompt size is the main storage and write-amplification variable. Evidence packing
already bounds model context, and the same retention deadline used for conversation
turns bounds long-term table growth. Operators should monitor insert latency, table
size, expired-row cleanup, and PostgreSQL TOAST usage under production traffic.

The current write is intentionally completed before delivery. This favors audit
durability and preserves the existing failure semantics: a turn is not treated as
successfully processed unless its audit row is stored. If measured production
latency later requires decoupling, the next safe step is a durable logging queue or
an inbox/outbox-linked audit worker—not an untracked background task that can be
lost on process shutdown.

## Privacy and security

Production pseudonymization and content encryption are separate controls:

1. `PRIVACY_HMAC_KEY` pseudonymizes sender and message identifiers.
2. `CONVERSATION_ENCRYPTION_KEY` encrypts question, prompt, and answer content.
3. When pseudonymization is active, plaintext content columns contain `[redacted]`.
4. The encryption key is supplied by the application and is never stored in the
   database.
5. `expires_at` and `PostgresConversationStore.cleanup_expired` enforce the existing
   retention lifecycle.

The encryption key is now honored whenever configured, independently of whether
conversation-aware retrieval is enabled. This allows secure audit logging without
turning on conversational memory. Without an encryption key in a privacy-enabled
deployment, identifiers remain pseudonymized but content is intentionally not
recoverable from the audit row.

Prompts can contain user text, retrieved evidence, and recent conversation context,
so they must be treated as at least as sensitive as questions and answers. Do not
send these fields to application logs, metrics labels, or error-reporting tags.

## Migration and rollout

1. Back up the database according to the normal deployment process.
2. Deploy/apply migrations with `promisetracker-index --ensure-schema` or the normal
   migration runner. Migration 006 is additive and takes no table rewrite for its
   nullable columns.
3. Run `promisetracker-schema-check`. It now requires both `prompt` and
   `prompt_ciphertext`.
4. Configure `CONVERSATION_ENCRYPTION_KEY` wherever recoverable audit content is
   required. Keep the same key available to readers that decrypt conversation data.
5. Deploy the application. During a rolling deployment, old processes ignore the
   new nullable columns and new processes can write them.
6. Confirm new rows contain encrypted prompts for generated/cache-hit answers and
   `NULL` prompts for deterministic responses.

Rollback is application-compatible: older application versions ignore the added
nullable columns. The columns should normally be retained rather than dropped,
because dropping them is destructive and immutable migrations are not edited after
application.

## Verification

Automated coverage proves the following contracts:

- generated `AnswerResult` objects expose the exact prompt sent to the LLM;
- cache-hit results expose the prompt stored with the cached answer;
- the audit insert includes plaintext/redacted and encrypted prompt columns;
- migration 006 is discoverable and renders without unresolved placeholders;
- schema validation rejects a database missing `prompt_ciphertext`;
- legacy-schema fallback remains available.

Recommended release verification commands:

```bash
ruff check .
black --check .
pytest
promisetracker-schema-check
```

The final schema check requires a configured, migrated PostgreSQL instance and is
therefore an environment-level deployment check rather than a hermetic unit test.

## Operational queries

Use only in an appropriately authorized environment. These examples avoid selecting
sensitive content.

```sql
-- Volume and latency by channel/day
SELECT channel, date_trunc('day', created_at) AS day,
       count(*) AS turns, avg(latency_ms) AS average_latency_ms
FROM rag.conversation_turns
GROUP BY channel, day
ORDER BY day DESC, channel;

-- Prompt coverage without exposing prompt text
SELECT
  count(*) FILTER (WHERE prompt_ciphertext IS NOT NULL) AS encrypted_prompts,
  count(*) FILTER (WHERE prompt IS NULL AND prompt_ciphertext IS NULL) AS no_llm_prompt,
  count(*) AS total_turns
FROM rag.conversation_turns;

-- Retention backlog
SELECT count(*) AS expired_rows
FROM rag.conversation_turns
WHERE expires_at <= now();
```

## Known boundaries and next steps

- Delivery-provider success/failure is not stored on the conversation row; delivery
  state remains the responsibility of the outbox.
- Failed pipeline attempts do not yet create audit rows because no completed response
  exists. Failure telemetry remains in structured application logs.
- Prompt bodies are intentionally not searchable. Quality-analysis exports should
  decrypt a bounded, access-controlled sample rather than add a broad text index.
- At substantially higher write volume, partitioning by `created_at`, scheduled
  retention cleanup, and a durable audit queue should be evaluated from measured
  database evidence.
