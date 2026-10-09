# Promise Tracker — Data Flows

Accurate as of: 2026-09-11
Scope: technical data exchange across the public website, the Laravel API, MySQL, queue workers, and external services. This is the dedicated data-flow layer referenced by the [architecture](architecture.md).

**Chatbot boundary:** the web chatbot widget is documented here up to `chat.promisetracker.ng`. The WhatsApp channel, the chat service's internals, and how it accesses promise data are owned by a **separate chatbot project** and are intentionally out of scope for this document.

## Data exchange summary

| # | Flow | Source → Destination | Contract | Payload | Sensitivity |
|---|---|---|---|---|---|
| 1 | Tracker read | Website → API → MySQL | `GET /presidential` etc. ([public API](api/public.md)) | Meter, promises, status counts | Public |
| 2 | Pledge submission | Website → API → MySQL | `POST /pledges` | Name, phone, email, state, LGA, PVC status, consent | **Voter PII** |
| 3 | Nomination emails | API → queue → worker → Mailtrap | `POST /pledges/{id}/nominations` | Nominee email addresses | **Contact PII** |
| 4 | Meter subscription | Website → API → MySQL | `POST /subscribe/to/meter` | Email, meter id | Contact PII |
| 5 | Admin login | Admin SPA → API → MySQL | `POST /api/v1/admin/authenticate` | Credentials; Sanctum token issued | Credentials |
| 6 | Promise publish | Admin SPA → API → MySQL | `POST promise/create`, `GET promise/publish/{id}` | Promise record, `published_status` | Public content |
| 7 | Media transcription | Admin SPA → API → queue → Google Cloud / Cloudinary | Resource-management uploads | Audio/video, transcripts | Internal |
| 8 | Chatbot (web widget) | Website → external chat service | `POST chat.promisetracker.ng/chat/web` | Visitor messages | External — see boundary note |

## Sequence diagrams

### 1. Public tracker read

```mermaid
sequenceDiagram
    actor Citizen
    participant W as Public website (React)
    participant API as Laravel API
    participant S as PromiseService
    participant DB as MySQL

    Citizen->>W: Opens tracker page
    W->>API: GET /presidential
    API->>S: currentPromises("presidential")
    S->>DB: SELECT current meter, published promises, status counts
    DB-->>S: Rows
    S-->>API: MeterData
    API-->>W: 200 { success, data, message }
    W-->>Citizen: Renders meter, promises, statuses
```

### 2. Voter pledge submission (PII)

```mermaid
sequenceDiagram
    actor Citizen
    participant W as Public website
    participant API as VoterPledgeController
    participant DB as MySQL

    Citizen->>W: Submits pledge form
    W->>API: POST /pledges (JSON)
    API->>API: Normalise name/email, validate unique identity
    API->>DB: INSERT INTO voter_pledges
    API-->>W: 201 { count, pledge_id }
    W-->>Citizen: Confirmation + nomination step
```

### 3. Nomination emails (queued)

```mermaid
sequenceDiagram
    actor Citizen
    participant W as Public website
    participant API as VoterPledgeController
    participant Q as DB queue (mail-high)
    participant Wk as mail-high worker
    participant MT as Mailtrap

    Citizen->>W: Enters friend emails
    W->>API: POST /pledges/{id}/nominations
    API->>API: Validate (max 5/request, 10 unique/pledge)
    API->>Q: Store nominations, dispatch SendVoterPledgeNomination
    API-->>W: 202 { accepted, duplicates }
    Wk->>Q: Pick job
    Wk->>MT: Send invitation email
    Wk->>Q: Update sent_at
```

### 4. Meter subscription

```mermaid
sequenceDiagram
    actor Visitor
    participant W as Public website
    participant API as MeterSubscribers
    participant DB as MySQL

    Visitor->>W: Enters email on a meter page
    W->>API: POST /subscribe/to/meter
    API->>API: Validate email + meter_id
    API->>DB: UPSERT meter_subscribers
    API-->>W: 200 (echoed data)
```

### 5. Admin login (Sanctum)

```mermaid
sequenceDiagram
    actor Staff
    participant SPA as Vue admin SPA
    participant API as Admin API
    participant G as admin guard
    participant DB as MySQL

    Staff->>SPA: Enters credentials
    SPA->>API: POST /api/v1/admin/authenticate
    API->>G: attempt(email, password)
    G->>DB: Verify admins row + hash
    G-->>API: Authenticated admin
    API->>DB: Create personal_access_token
    API-->>SPA: 200 { user, access_token }
    SPA->>SPA: Store token, attach Bearer header
```

### 6. Promise publish (admin workflow)

```mermaid
sequenceDiagram
    actor Staff
    participant SPA as Vue admin SPA
    participant API as Admin promise API
    participant DB as MySQL

    Staff->>SPA: Creates promise
    SPA->>API: POST promise/create
    API->>DB: INSERT promises (published_status = in_review)
    Staff->>SPA: Publishes promise
    SPA->>API: GET promise/publish/{id}
    API->>DB: UPDATE published_status = published
    Note over API,DB: Public tracker reads now include the promise
```

### 7. Media transcription (jobs)

```mermaid
sequenceDiagram
    actor Staff
    participant SPA as Vue admin SPA
    participant API as Resource management API
    participant Q as DB queue (default)
    participant G as Google Cloud
    participant C as Cloudinary
    participant DB as MySQL

    Staff->>SPA: Uploads course audio/video
    SPA->>API: POST resource-management/upload
    API->>C: Store media
    API->>Q: Dispatch TranscribeAudio
    Q->>G: Fetch + transcribe via Storage/Speech
    G-->>Q: Transcript
    Q->>DB: Store transcription text/data
```

### 8. Chatbot web widget (external boundary)

```mermaid
sequenceDiagram
    actor Visitor
    participant W as Public website (ChatbotWidget)
    participant CS as chat.promisetracker.ng

    Visitor->>W: Types a message
    W->>CS: POST /chat/web
    CS-->>W: Bot reply
    Note over CS: WhatsApp channel + chat-service internals +<br/>promise-data access are documented in the<br/>separate chatbot architecture project
```

## Storage and queue notes

- All writes go through the Laravel API; no client writes to MySQL directly.
- Queued jobs live in the `jobs` table; the `default` and `mail-high` queues each have a dedicated systemd worker (see [deployment](deployment/README.md)).
- Public reads are filtered by `published_status = published` (promises) and soft-delete exclusion; see the [data model](data/README.md).
