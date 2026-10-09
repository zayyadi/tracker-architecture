# Data Model — Promise Tracking

Accurate as of: 2026-09-10

## Entity relationship diagram

```mermaid
erDiagram
    ADMINS ||--o{ PROMISE_TRACKERS : creates
    ADMINS ||--o{ METERS : creates
    ADMINS ||--o{ PROMISES : authors
    PROMISE_TRACKERS ||--o{ METERS : has
    COUNTRIES ||--o{ STATES : has
    STATES ||--o{ METERS : scopes
    PROMISE_TRACKERS ||--o{ PROMISES : categorises
    METERS ||--o{ PROMISES : contains
    PROMISES ||--o{ UPDATES : receives

    PROMISE_TRACKERS {
        uuid id PK
        string name
        text description
        enum type "presidential|state|legislative"
        bigint admin_id FK
    }
    METERS {
        uuid id PK
        uuid promise_tracker_id FK
        string name "official's name"
        uuid state_id FK "nullable"
        string chamber "nullable"
        string meter_name
        enum meter_type "promise_tracker|promise_ring"
        date tenure_date_start
        date tenure_date_end
        string meter_img "nullable"
        bigint admin_id FK
    }
    PROMISES {
        uuid id PK
        uuid meter_id FK
        uuid promise_tracker_id FK
        enum status "not_yet_rated|in_the_works|stalled|compromised|kept|broken"
        string title
        string source
        string subject_area "nullable"
        text promise_blog
        enum priority "high|medium|low"
        string promise_img "nullable"
        text description
        date promise_start_date
        date promise_end_date
        bigint admin_id FK
        int update_count
        datetime last_updated_at "nullable"
        enum published_status "in_review|published|rejected"
    }
    UPDATES {
        uuid id PK
        uuid promise_id FK
        text blog
        enum update_type "update|reminder"
        string promise_tracker_type "nullable"
        boolean is_published
        string status "nullable"
        string published_status "nullable"
    }
    COUNTRIES {
        uuid id PK
        string name
        string iso_code
        string dialing_code "nullable"
    }
    STATES {
        uuid id PK
        string name
        uuid country_id FK
        string iso_code "nullable"
        string state_map "nullable"
    }
```

**None of the relationships above are enforced by database foreign keys** — all `FK` markers are conventional. Orphaned rows are possible; the service layer is the only guard.

## Key tables

| Table | Purpose | Notable columns |
|---|---|---|
| `promise_trackers` | Top-level tracker per office type | `type` enum; one presidential, state, or legislative tracker owns many meters |
| `meters` | A rated "meter" for an official or body | `meter_type` distinguishes tracker meters from Promise Ring meters; `state_id`/`chamber` scope it; tenure dates pick the *current* presidential meter |
| `promises` | A single tracked promise | Six-value `status`; `published_status` gates public visibility (`in_review` default, `published`, `rejected`) |
| `updates` | Promise updates and reminders | `update_type` splits updates from reminders; `promise_tracker_type` powers the public reminders feed |

## Invariants the application relies on

- A promise is public only when `published_status = published` and not soft-deleted.
- The *current* presidential meter is the one whose tenure dates bracket today.
- Promise Ring meters are `meter_type = promise_ring` scoped to a state; tracker meters are `promise_tracker`.
- Status counts on the public site are computed from published promises only.
