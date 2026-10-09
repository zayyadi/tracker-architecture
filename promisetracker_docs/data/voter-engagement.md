# Data Model — Voter Engagement (PII)

Accurate as of: 2026-09-10
Classification: **Highest-sensitivity PII** — full names, phone numbers, emails, state/LGA, PVC status, and consent flags.

## Entity relationship diagram

```mermaid
erDiagram
    VOTER_PLEDGES ||--o{ VOTER_PLEDGE_NOMINATIONS : invites
    VOTER_PLEDGES |o--o{ PLEDGE_REMINDER_CONTACTS : reminds
    METERS ||--o{ METER_SUBSCRIBERS : "subscribed to"

    VOTER_PLEDGES {
        uuid id PK
        string full_name
        string normalized_full_name UK
        string phone_number UK
        string email "nullable"
        string normalized_email UK "nullable"
        string state
        string lga
        string pvc_status "yes|no|processing"
        boolean accepted_pledge
        boolean wants_updates
    }
    VOTER_PLEDGE_NOMINATIONS {
        uuid id PK
        uuid voter_pledge_id FK "enforced, cascade"
        string email
        timestamp queued_at "nullable"
        timestamp sent_at "nullable"
    }
    PLEDGE_REMINDER_CONTACTS {
        uuid id PK
        uuid voter_pledge_id FK "nullable, conventional"
        string name
        string email "nullable"
        string phone_number UK
        string state
        string lga
        timestamp last_reminded_at "nullable"
    }
    METER_SUBSCRIBERS {
        uuid id PK
        string email
        uuid meter_id FK "conventional"
    }
```

## Key tables

| Table | Purpose | Integrity notes |
|---|---|---|
| `voter_pledges` | A citizen's pledge-to-vote submission | Unique on `phone_number`, `normalized_full_name`, and `normalized_email` (nullable). Normalisation trims, collapses whitespace, and lower-cases. |
| `voter_pledge_nominations` | Friends a pledger invited | **Only enforced FK in this domain** (`voter_pledge_id`, cascade delete). Unique `(voter_pledge_id, email)`; `queued_at`/`sent_at` track delivery. Max 10 unique emails per pledge enforced in the controller. |
| `pledge_reminder_contacts` | Reminder contact roster | `voter_pledge_id` is nullable and **not** a foreign key; `phone_number` is unique; state/LGA indexed. |
| `meter_subscribers` | Email subscribers to a meter | **No unique `(email, meter_id)`** — dedup is application-level via `updateOrCreate` only. |

## Privacy handling

- These tables are written by public endpoints but never exposed by public reads.
- Consent is explicit: `accepted_pledge` must be true; `wants_updates` gates future contact.
- Nominations store only an email address, with delivery state tracked for the queue.
- Any export, analytics, or admin viewing of these tables is a PII access event and should be logged.
