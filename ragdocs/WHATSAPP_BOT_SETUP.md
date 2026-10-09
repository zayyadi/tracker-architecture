# WhatsApp Bot Setup and Customization Guide

This guide connects the PromiseTracker RAG chatbot to the official Meta
WhatsApp Business Platform Cloud API. It covers a Meta test number, a real
business number, local development, production deployment, verification,
customization, and troubleshooting.

The application already implements the required endpoints and Cloud API client:

```text
WhatsApp user
    -> Meta WhatsApp Cloud API
    -> HTTPS POST /webhook/whatsapp
    -> PostgreSQL durable inbox
    -> PromiseTracker worker and RAG pipeline
    -> Meta Graph API /{PHONE_NUMBER_ID}/messages
    -> WhatsApp user
```

The relevant implementation files are:

- `app/main.py`: webhook verification, inbound admission, and processing
- `app/whatsapp.py`: payload parsing and outbound Cloud API requests
- `app/worker_cli.py`: durable inbox processing and outbox delivery
- `app/settings.py`: environment settings and production validation
- `app/prompt.py`: bot identity, response rules, greeting, and WhatsApp length
- `.env`: local runtime configuration; never commit its secrets

## 1. What you need

Before starting, have:

- A Meta developer account and a Meta Business Portfolio
- A Meta app with the WhatsApp product added
- A WhatsApp Business Account (WABA)
- A Meta test number or a production phone number
- A public HTTPS URL that forwards to the FastAPI service
- The base RAG stack configured and working as described in
  `docs/STARTUP_GUIDE.md`
- PostgreSQL, Qdrant, the BGE model service, FastAPI, and the durable worker
  running for the normal production path

Meta must be able to reach the callback over public HTTPS. It cannot call
`127.0.0.1`, a private LAN address, or a URL with an invalid TLS certificate.

## 2. Create the Meta app and test setup

Meta changes dashboard labels occasionally, but the flow is generally:

1. Open [Meta for Developers](https://developers.facebook.com/apps/) and create
   an app associated with the correct Business Portfolio.
2. Select a business-oriented app use case, then add the **WhatsApp** product.
3. Open **WhatsApp > API Setup** in the app dashboard.
4. Use Meta's test phone number for the first test, or add the intended business
   number.
5. Add and verify your own WhatsApp number as a test recipient if the app is
   still in development/test mode.
6. Record these values:

   - **Phone number ID**: use for `WHATSAPP_PHONE_NUMBER_ID`
   - **WhatsApp Business Account ID**: useful for Meta administration, but it is
     not currently an application environment variable
   - **Temporary access token**: suitable only for an initial test
   - **App secret** from **App settings > Basic**: use for
     `WHATSAPP_APP_SECRET`

Do not confuse the visible WhatsApp phone number, WABA ID, Meta app ID, and
phone number ID. This application sends to
`/{WHATSAPP_PHONE_NUMBER_ID}/messages`, so it needs the **phone number ID**.

## 3. Create the four WhatsApp settings

Generate a strong webhook verify token locally. This is a secret string chosen
by you; Meta does not generate it:

```bash
openssl rand -hex 32
```

Add the following to `rag/.env`:

```dotenv
# A random value chosen by you and entered identically in the Meta webhook UI.
WHATSAPP_VERIFY_TOKEN=replace-with-a-random-verify-token

# Temporary dashboard token for a short test; use a System User token in production.
WHATSAPP_ACCESS_TOKEN=replace-with-the-cloud-api-access-token

# Numeric ID shown beside the sending number in WhatsApp > API Setup.
WHATSAPP_PHONE_NUMBER_ID=replace-with-the-phone-number-id

# The secret of the Meta app that owns this WhatsApp integration.
WHATSAPP_APP_SECRET=replace-with-the-meta-app-secret

# Optional. Keep the project default unless deliberately upgrading Graph API.
WHATSAPP_GRAPH_BASE_URL=https://graph.facebook.com/v20.0
```

Rules for these values:

- Do not add quotes unless they are intentionally part of a value.
- Do not add `export` in `.env` or a systemd `EnvironmentFile`.
- Never put the access token or app secret in frontend code, logs, screenshots,
  source control, or webhook URLs.
- Restart both API and worker after changing `.env`; each process loads its own
  settings.
- `WHATSAPP_VERIFY_TOKEN` authenticates Meta's one-time webhook GET challenge.
  `WHATSAPP_APP_SECRET` validates every inbound POST signature. They are not
  interchangeable.

The code currently defaults to Graph API `v20.0`. Graph API versions are
retired over time. Before production launch, select a Meta-supported version,
test it, and set `WHATSAPP_GRAPH_BASE_URL` consistently instead of silently
depending on the default.

## 4. Start the local application

Complete the database migration and indexing steps in `docs/STARTUP_GUIDE.md`
first. Then run the API and worker in separate terminals:

```bash
cd /opt/usr/devs/rag
source .venv/bin/activate
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 1
```

```bash
cd /opt/usr/devs/rag
source .venv/bin/activate
python -m app.worker_cli
```

The normal setting is:

```dotenv
DURABLE_INGRESS=true
RATE_LIMIT_BACKEND=postgres
MESSAGE_PROCESSING_CONCURRENCY=4
```

With durable ingress enabled, FastAPI accepts and persists the webhook quickly;
the worker generates and sends the reply. If the API is running without the
worker, inbound messages can be accepted but no reply will arrive.

Check readiness:

```bash
curl -sS http://127.0.0.1:8000/health
curl -sS http://127.0.0.1:8000/ready
```

Use the exact health paths returned by the current application if these are
changed later. `/ready` should not report failed database or retrieval state.

## 5. Expose the webhook over HTTPS

For local development, run an HTTPS tunnel that forwards to
`http://127.0.0.1:8000`. For production, use a stable domain and reverse proxy,
for example:

```text
https://bot.example.org/webhook/whatsapp
                 -> http://127.0.0.1:8000/webhook/whatsapp
```

Requirements:

- The public URL must preserve the request body and the
  `X-Hub-Signature-256` header.
- Do not rewrite POST bodies before FastAPI receives them; signature validation
  is calculated over the original bytes.
- Allow both `GET` and `POST` on `/webhook/whatsapp`.
- Do not put basic authentication, a login page, or an IP-only restriction in
  front of this route.
- Forward the original query string during webhook verification.
- Use a stable production URL. A temporary tunnel URL must be re-entered in Meta
  whenever it changes.

Test the local GET verification handler before configuring Meta:

```bash
curl -i \
  "http://127.0.0.1:8000/webhook/whatsapp?hub.mode=subscribe&hub.verify_token=YOUR_VERIFY_TOKEN&hub.challenge=12345"
```

Expected result: HTTP `200` with body `12345`. A wrong token should return
HTTP `403`.

Repeat that test through the public HTTPS URL. Do not continue until the public
request reaches this application successfully.

## 6. Configure the Meta webhook

In the Meta app dashboard:

1. Open the WhatsApp product's **Configuration** page (or the Webhooks page for
   the **WhatsApp Business Account** object).
2. Choose **Configure a webhook** or **Edit callback URL**.
3. Set the callback URL to:

   ```text
   https://bot.example.org/webhook/whatsapp
   ```

4. Enter the exact value of `WHATSAPP_VERIFY_TOKEN` as the verify token.
5. Save/verify the callback.
6. Subscribe the app to the **messages** webhook field.
7. Confirm the app is subscribed to the correct WABA/phone-number setup, not a
   different test account in the same Business Portfolio.

Meta sends a GET challenge during step 5. Normal incoming messages are POSTed
later with an `X-Hub-Signature-256: sha256=...` signature. The application
rejects missing or invalid signatures with HTTP `401`; a plain hand-written
`curl` POST without a correctly calculated signature is therefore expected to
fail.

The official payload reference shows that WhatsApp webhook events use the
`whatsapp_business_account` object and the `messages` field:
[Meta WhatsApp webhook payload reference](https://www.postman.com/meta/whatsapp-business-platform/folder/tduohwq/webhook-payload-reference).

## 7. Send the first message

There are two useful independent tests.

### Test outbound Cloud API credentials

Replace the placeholders and use a recipient in international format. For the
Meta test number, the recipient must first be added to the allowed test list.

```bash
curl -i -X POST \
  "https://graph.facebook.com/v20.0/PHONE_NUMBER_ID/messages" \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "messaging_product": "whatsapp",
    "to": "RECIPIENT_NUMBER_WITH_COUNTRY_CODE",
    "type": "text",
    "text": {"body": "PromiseTracker WhatsApp connection test"}
  }'
```

This tests the token, phone number ID, recipient, and Meta-side permissions. It
does not test the inbound webhook or RAG pipeline.

### Test the complete bot

1. Send a normal text message from an allowed WhatsApp user to the Meta test or
   production business number.
2. Watch the API and worker logs.
3. Confirm the user receives a grounded PromiseTracker answer.
4. Ask a query that returns multiple promises and test the **View summaries**
   interactive list.
5. Select a list row and confirm its summary arrives.

For the current systemd setup:

```bash
tail -f /opt/usr/devs/rag/log/api.log
```

```bash
tail -f /opt/usr/devs/rag/log/worker.log
```

The webhook returns `{"status":"accepted"}` even for delivery/status payloads
that contain no supported user message. That is normal.

## 8. Replace the temporary token for production

The token displayed on the API Setup page is intended for initial testing and
expires. For production, create a Business Portfolio System User token:

1. Open Meta **Business settings > Users > System users**.
2. Create a system user owned by the same Business Portfolio.
3. Assign the relevant Meta app and WhatsApp Business Account assets to it with
   the access needed to manage/send messages.
4. Generate a token for the correct app with at least
   `whatsapp_business_messaging`; Meta administration flows may also require
   `whatsapp_business_management`.
5. Store it in a secret manager or the protected server environment file and
   update `WHATSAPP_ACCESS_TOKEN`.
6. Restart API and worker and repeat the outbound and end-to-end tests.

Treat a System User token as a password even when it is described as permanent:
it can still be revoked or invalidated by security and asset changes. Document
its owning app, System User, WABA, permissions, creation date, and rotation
procedure.

## 9. Move from a test number to a business number

For production, finish the current Meta onboarding requirements shown in the
Business Manager/dashboard. Typically this includes:

- Adding and verifying a phone number that can receive the verification code
- Setting the WhatsApp business display name and business profile
- Ensuring the number is attached to the intended WABA
- Adding billing/payment details where Meta requires them
- Completing business verification and app review/access requirements that
  apply to the intended scale and use case
- Moving the app to Live mode when appropriate
- Replacing the test phone number ID in `WHATSAPP_PHONE_NUMBER_ID`
- Confirming the production WABA is subscribed to the app's webhook

Changing from the test number to a real number usually changes the phone number
ID. Restart both services after the `.env` update.

This code sends free-form answers in response to inbound messages. It does not
currently implement outbound message templates. Observe Meta's current
customer-service-window and template rules: proactive or out-of-window messages
require an approved template and additional application support.

## 10. Production configuration

WhatsApp is only one part of the production validation. A representative
channel/runtime block is:

```dotenv
APP_ENV=production

WHATSAPP_VERIFY_TOKEN=replace-with-random-verify-token
WHATSAPP_ACCESS_TOKEN=replace-with-system-user-token
WHATSAPP_PHONE_NUMBER_ID=replace-with-production-phone-number-id
WHATSAPP_APP_SECRET=replace-with-meta-app-secret
WHATSAPP_GRAPH_BASE_URL=https://graph.facebook.com/v20.0

PRIVACY_HMAC_KEY=replace-with-an-independent-high-entropy-secret
PRIVACY_KEY_VERSION=privacy-v1
DURABLE_INGRESS=true
RATE_LIMIT_BACKEND=postgres
MESSAGE_PROCESSING_CONCURRENCY=4
RATE_LIMIT_PER_MINUTE=20
ACTIVE_INDEX_RETRIEVAL_ENABLED=true
```

Also configure the database, SSL mode, LLM provider, embedding service, Qdrant,
reranker, and index settings from `deploy/systemd/app.env.example` and
`docs/STARTUP_GUIDE.md`. In production, `app/settings.py` rejects startup if
required settings are missing, database SSL is disabled, durable ingress is
off, or active-index retrieval is disabled.

With the repository's systemd units:

```bash
sudo systemctl restart promisetracker-api.service promisetracker-worker.service
sudo systemctl status promisetracker-api.service promisetracker-worker.service
```

Do not enable the Telegram poller merely to run WhatsApp. The WhatsApp path
needs the API and worker, plus their PostgreSQL/Qdrant/model dependencies.

## 11. Customize the bot

Make one customization at a time, run the focused tests, restart API and worker,
then perform an end-to-end WhatsApp test.

### Bot identity, tone, and answer format

Edit `SYSTEM_PROMPT` and the task instructions in `app/prompt.py`. This is where
the current **Mai Gaskiya** identity, evidence-only policy, official rating
labels, tone, and answer sections are defined.

Keep the evidence and citation constraints unless product requirements explicitly
change. They prevent the model from presenting unsupported political claims as
PromiseTracker facts.

### Greeting and fallback response

- Greeting detection and greeting text: `is_greeting()` and
  `format_greeting_answer()` in `app/prompt.py`
- Failure/no-answer response: `FALLBACK_RESPONSE` in `app/main.py`

When adding greeting phrases or languages, extend the greeting tests as well as
the code so ordinary questions are not accidentally classified as greetings.

### WhatsApp answer length

`trim_for_whatsapp()` in `app/prompt.py` currently limits rendered answers to
3,500 characters and appends `Reply MORE for details.` when truncated. Change
the limit or suffix there. Confirm that the `MORE` behavior is supported by the
conversation/query logic before promising it in user-facing copy.

### Interactive promise summaries

`WhatsAppClient._send_interactive_list()` in `app/whatsapp.py` controls:

- The body text: `Choose a promise to view its summary.`
- The button label: `View summaries`
- The section title: `Promises`
- The row description
- Grouping into batches of 10 rows

`_whatsapp_title()` normalizes and truncates each row title to 24 characters to
fit the WhatsApp list format. Preserve Meta's current interactive-message limits
when changing labels, section sizes, or title lengths.

The callback/action identifiers are created by
`build_summary_actions()` in `app/interactions.py` and parsed by
`parse_summary_request()`. Keep identifiers stable and opaque; do not put
credentials or sensitive user data in them.

### Models and answer quality

Customize provider/model selection in `.env`, not in the WhatsApp client:

```dotenv
LLM_PROVIDER=gemini
GEMINI_DEFAULT_MODEL=gemini-3-flash-preview
GEMINI_ESCALATION_MODEL=gemini-2.5-pro
```

The repository also supports the provider settings defined in
`app/settings.py`. Re-run evaluation and retrieval tests after changing models,
prompts, embedding models, score thresholds, top-k values, or the reranker.

### Conversation memory

Conversation memory is channel-aware and optional:

```dotenv
CONVERSATION_ENABLED=true
CONVERSATION_REWRITE_ENABLED=true
CONVERSATION_ENCRYPTION_KEY=replace-with-an-independent-high-entropy-secret
CONVERSATION_MAX_TURNS=8
CONVERSATION_TTL_MINUTES=30
```

Use a key independent of the Meta secrets and `PRIVACY_HMAC_KEY`. Enabling
memory changes data-retention/privacy behavior, so set policy and consent
expectations before production use.

### Rate limits and concurrency

```dotenv
RATE_LIMIT_PER_MINUTE=20
MESSAGE_PROCESSING_CONCURRENCY=4
```

The webhook rate limit is keyed by the sender's WhatsApp number. With PostgreSQL
backing, limits work across processes. Raise concurrency only after checking
database, model service, LLM, Graph API, and host capacity.

### Add media, location, audio, or template messages

The current parser accepts only:

- Inbound `text` messages
- Inbound interactive `button_reply` and `list_reply` actions

Images, documents, voice notes, video, contacts, locations, reactions, and other
message types are currently ignored. To add one:

1. Add a typed inbound model in `app/models.py`.
2. Parse the exact webhook shape in `app/whatsapp.py`.
3. Normalize it into the internal `ChatTextMessage` or introduce a carefully
   designed richer internal message type.
4. If media is involved, fetch it from Meta securely, enforce size/MIME limits,
   scan it as appropriate, and define deletion/retention behavior.
5. Add outbound client methods if the reply is not plain text or an interactive
   list.
6. Add parser, signature, worker, retry, and end-to-end tests.

For proactive or out-of-window messaging, add a dedicated template-sending
method instead of trying to send free-form `text` payloads.

## 12. Security and privacy checklist

- Keep `WHATSAPP_APP_SECRET` configured. The code fails closed when it is absent.
- Preserve raw request bytes until `verify_meta_signature()` validates them.
- Keep the webhook endpoint HTTPS-only on the public internet.
- Restrict access to `.env`; do not commit it.
- Use separate secrets for webhook verification, Meta app signing, privacy
  pseudonymization, and conversation encryption.
- Rotate a credential immediately if it appears in logs, shell history,
  screenshots, tickets, or commits.
- Keep production and development Meta apps/WABAs/secrets separate where
  practical.
- Limit Meta System User asset access and permissions to what the integration
  needs.
- Review query/response logging and retention behavior in
  `docs/query-response-logging.md` before handling real users.
- Do not log full access tokens, app secrets, message bodies, or phone numbers.
- Back up PostgreSQL and define retention/deletion procedures for inbox,
  outbox, conversations, and interaction logs.

Meta signs webhook notifications with HMAC SHA-256 in
`X-Hub-Signature-256`; the implementation verifies that header in
`app/security.py`. Meta's general signature guidance is available in its
[official API collection](https://www.postman.com/meta/messenger-platform-api/folder/22794852-b5d97624-14d8-4e67-a2e4-529add49ca58).

## 13. Troubleshooting

### Meta says the callback URL or verify token could not be validated

- Test the public GET URL with `curl`, not only the local URL.
- Confirm callback path is exactly `/webhook/whatsapp`.
- Confirm the Meta verify token exactly matches `WHATSAPP_VERIFY_TOKEN`.
- Check HTTPS certificate, DNS, proxy routing, query-string forwarding, and API
  logs.
- Restart the API after editing `.env`.

### POST returns `401 Invalid signature`

- `WHATSAPP_APP_SECRET` is missing, copied incorrectly, or belongs to another
  Meta app.
- The reverse proxy or middleware changed the body.
- A manual test omitted or incorrectly computed `X-Hub-Signature-256`.
- The request is not a genuine notification from the configured Meta app.

Do not disable signature checking to make this error disappear.

### Meta receives `200`, but the bot does not reply

- Check `worker.log`; with `DURABLE_INGRESS=true`, the worker sends the reply.
- Confirm the worker and API use the same `.env` and PostgreSQL database.
- Check database migrations and `/ready`.
- Confirm Qdrant, the model service, LLM credentials, and the active index are
  healthy.
- Confirm the inbound event is text or a supported interactive reply.
- Check rate-limit messages in `api.log`.
- Look for Graph API errors in `worker.log`.

### Graph API returns `401`, error `190`, or an expired-token error

Replace the temporary/revoked token, verify the System User and assigned assets,
restart both services, and rerun the direct outbound API test.

### Graph API returns a permission or unsupported request error

- Verify the token has `whatsapp_business_messaging`.
- Verify the System User can access the correct app and WABA.
- Verify `WHATSAPP_PHONE_NUMBER_ID` is a phone number ID, not the WABA, app ID,
  or the visible phone number.
- Check that `WHATSAPP_GRAPH_BASE_URL` uses a currently supported version and
  test the upgrade against Meta's current documentation.

### A test recipient never receives a message

- In test mode, add and verify the recipient in Meta's API Setup page.
- Use the country code and digits expected by Meta.
- Confirm the recipient is testing the correct Meta test number.
- Test sending with direct `curl` to isolate Meta credentials from the bot.

### Incoming media gets no answer

This is expected in the current implementation. Only inbound text and
interactive list/button replies are parsed.

### The API starts locally but fails in production

`APP_ENV=production` enables stricter validation. Read the startup exception and
configure all required database, WhatsApp, privacy, LLM, embedding, durable
ingress, SSL, and active-index settings. See `app/settings.py` and
`deploy/systemd/app.env.example` for the authoritative list.

## 14. Verification checklist

Before declaring the bot ready:

- [ ] Local `/health` and `/ready` checks pass
- [ ] Public HTTPS GET challenge returns the supplied challenge
- [ ] Wrong verify token returns `403`
- [ ] Unsigned/invalid webhook POST returns `401`
- [ ] Meta webhook is subscribed to the `messages` field for the correct WABA
- [ ] Direct outbound Cloud API text reaches a verified recipient
- [ ] Inbound text reaches `api.log` and the durable inbox
- [ ] Worker processes the message and sends an answer
- [ ] Interactive promise list and selected summary both work
- [ ] Duplicate webhook delivery does not create duplicate logical processing
- [ ] Restarting the worker processes queued messages
- [ ] Production uses a System User token, not a temporary dashboard token
- [ ] Production phone number ID replaces the test number ID
- [ ] Secrets, logging, privacy, backups, and retention have been reviewed
- [ ] Prompt/model/retrieval changes pass tests and an end-to-end WhatsApp test

## 15. Focused developer checks

After WhatsApp or shared message-processing changes, run at least:

```bash
cd /opt/usr/devs/rag
source .venv/bin/activate
pytest -q \
  tests/test_core.py \
  tests/test_summary_channels.py \
  tests/test_message_processing.py \
  tests/test_durable_runtime.py \
  tests/test_worker.py \
  tests/test_worker_cli.py \
  tests/test_settings.py
```

Then restart the API and worker and repeat the direct outbound plus complete
inbound WhatsApp tests. Unit tests cannot confirm the Meta app, token, WABA,
phone number, webhook subscription, public proxy, or live delivery setup.

