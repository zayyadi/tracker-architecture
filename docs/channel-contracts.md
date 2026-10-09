# Shared channel contracts

Design agreed: 2026-10-09. Status: proposed contract, not a verified existing API.

The documents identify `POST /chat/web` but do not establish its complete browser schema. Inspect the deployed widget and FastAPI route first. Preserve the old response shape through a compatibility adapter or versioned route; do not silently replace synchronous replies with job admission.

## Browser sessions and jobs

Both browser surfaces use one thin client/renderer with separate sessions. Serve the standalone client at `chat.promisetracker.ng/`, with APIs under that origin. A same-origin proxy on the public website can simplify embedded access. For direct cross-origin access, allow only intended origins and use a scoped anonymous session token. Never ship provider/database credentials to browsers.

Proposed additive routes:

| Route | Behavior |
|---|---|
| `POST /v1/chat/sessions` | Establish short-lived opaque session; return scoped bearer token and expiry |
| `POST /v1/chat/jobs` | Validate and durably admit a question; return HTTP 202 and job ID |
| `GET /v1/chat/jobs/{id}` | Authorised status/result read; conditional responses supported |

Tokens stay in memory/session storage, never URLs/logs. Enforce server-side job ownership, expiry, session admission controls, and rate limits. This is session access, not verified citizen identity. Never trust a client-supplied sender ID.

Example job request:

```json
{
  "client_message_id": "random-uuid-preserved-on-retry",
  "message": "What is the current status of this promise?",
  "locale": "en-NG"
}
```

Validate message length, empty text, supported locale, and configured request-size bounds. Initial language capability must be evaluated; additional language support is a later scope decision.

After database commit, return:

```json
{
  "job_id": "opaque-id",
  "state": "queued",
  "status_url": "/v1/chat/jobs/opaque-id",
  "retry_after_seconds": 2
}
```

States are `queued`, `processing`, `completed`, and `failed`. Example completed result:

```json
{
  "job_id": "opaque-id",
  "state": "completed",
  "outcome": "answer",
  "answer": "The official rating is Kept. [E1]",
  "citations": [{"id": "E1", "promise_id": "source-id", "url": "https://promisetracker.ng/verified-source-path", "title": "Promise title"}],
  "freshness": {
    "index_version": "active-version",
    "source_watermark_at": "2026-10-09T12:00:00Z",
    "generated_at": "2026-10-09T12:01:00Z",
    "state": "fresh"
  }
}
```

IDs/URLs are illustrative; map actual public routes before implementation. Outcomes include `answer`, `clarification`, `abstain`, and nonfactual greeting. Freshness is `fresh`, `delayed`, or `unknown`, evaluated at read time. Render escaped text/sanitised markup and validated citation links, never arbitrary model HTML.

Idempotency is scoped to `(channel, session subject, client_message_id)`. Same key/content returns the original job; conflicting content returns 409. Enforce a unique constraint and atomic admission. Reconnection retries preserve the message ID; disconnection does not cancel admitted work.

Poll only for pending work while chat is open, honour server guidance, back off with jitter, pause offline, and resume the same job. Suggested intervals are 2, 4, then 8 seconds. Show queued/processing feedback. End automatic polling after a bounded wait and retain job ID for explicit status checks; do not re-submit merely because the answer is slow. Streaming can be considered later after reconnection and weak-network testing.

| Response | Client action |
|---|---|
| 400/413 | Display validation issue; do not retry unchanged input |
| 401/403 | Handle expired/invalid session without revealing other jobs |
| 404 | Show unavailable job; avoid cross-session existence disclosure |
| 409 | Explain conflicting retry |
| 429 | Honour `Retry-After`; preserve input |
| 503 | Retry admission with the same message ID when instructed |
| Failed job | Safe error code and retryability; no raw provider error |

Use ETags and `Cache-Control: private, no-cache` for conditional status reads. Use `no-store` for session creation and sensitive mutations. Exclude chat traffic from shared CDN and public service-worker caching. Avoid URL/token leakage and output injection.

## WhatsApp and Telegram

| Concern | WhatsApp | Telegram |
|---|---|---|
| Verification | Provider signature over original request bytes | Webhook secret or authenticated poller |
| Deduplication | Verified sender + external message ID + channel | Verified sender/chat + update/message ID + channel |
| Admission | Commit inbox before acknowledging supported event | Commit before webhook acknowledgement or advancing polling offset |
| Output | Provider-supported text/links; safe splitting | Escaped provider formatting/links; safe splitting |
| Delivery | Durable outbox, bounded retry, provider IDs | Same; webhook and polling are mutually exclusive |

Ignore receipts/control events as user questions. Preserve sender ordering or sequence turns to avoid concurrent follow-up contamination. Verify exact provider length limits, credentials, and API policies against selected versions before launch.

Adapters render the same structured answer, not independently generated answers. Preserve ratings, freshness, and citations across chunks. Retry transient failures with backoff; send permanent failures to an operational dead-letter record. Redact recipient identifiers and tokens from logs.

## Acceptance checks

- [ ] Existing `/chat/web` schema recorded; migration compatibility verified.
- [ ] Sessions cannot access each other's jobs/history.
- [ ] Concurrent duplicate submissions create one job; conflicting content is rejected.
- [ ] Disconnect/reconnect resumes admitted work without duplicate model calls.
- [ ] Input bounds, rate limits, backpressure, and safe failures exercised.
- [ ] Every surface preserves evidence, ratings, citations, and freshness.
- [ ] Signed replay, worker restart, failed delivery, and Telegram offset recovery exercised.
- [ ] CORS, output escaping, access controls, and cache headers verified in staging.
