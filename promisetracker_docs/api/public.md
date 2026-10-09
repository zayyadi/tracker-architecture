# Public API Contract

Accurate as of: 2026-09-10
Status: **Inferred** from the React website, controllers, services, Form Requests, and tests.

Base URL: `https://admin.promisetracker.ng/api/v1/promise-tracker`
Auth: none. All reads and writes in this document are public.

## Endpoint index

| # | Method | Path | Purpose |
|---|---|---|---|
| 1 | GET | `/presidential` | Current presidential meter, promises, and status counts |
| 2 | GET | `/state/{state}` | Current meter for a state (by name) |
| 3 | GET | `/legislative/{chamber}` | Current meter for `upper-chamber` or `lower-chamber` |
| 4 | GET | `/meter/{meterId}/presidential` | Promises for a specific presidential meter |
| 5 | GET | `/meter/{meterId}/legislative/{chamber}` | Promises for a specific legislative meter |
| 6 | GET | `/view/promise/{promiseId}` | Promise detail with published updates |
| 7 | GET | `/latest/promises/fetch` | Three random published promises |
| 8 | GET | `/all/promise-ring/fetch/{state}` | Promise Ring promises and status counts for a state |
| 9 | GET | `/promise/reminders/fetch/{type}` | Published reminders by tracker type |
| 10 | GET | `/view/promise/reminder/{updateId}` | A single reminder update with its promise |
| 11 | GET | `/all/leaderboards/scores?type={all|this_week}` | Learner leaderboard |
| 12 | GET | `/leaderboard/top/student` | Top three active students |
| 13 | GET | `/total/promise/tracked` | Aggregate published-promise statistics |
| 14 | GET | `/get/cohort/text` | Homepage cohort text |
| 15 | GET | `/all/promise-ring/states` | States with Promise Ring meters and promise counts |
| 16 | POST | `/subscribe/to/meter` | Subscribe an email to meter updates |
| 17 | GET | `/pledges/count` | Total voter pledges |
| 18 | POST | `/pledges` | Create a voter pledge |
| 19 | POST | `/pledges/{pledgeId}/nominations` | Queue nomination invitation emails |
| 20 | GET | `/all/current/partners` | Enabled partners (historical — not called by current site code) |

## Shared shapes

### `MeterData` (endpoints 1–5)

```json
{
  "promises": [],
  "all_meters": [],
  "current_meter": {},
  "promise_count": 0,
  "not_rated": 0,
  "in_works": 0,
  "stalled": 0,
  "compromised": 0,
  "kept": 0,
  "broken": 0
}
```

`promises` contains published promises only. `current_meter` is the full meter record; `all_meters` is a reduced selection (`id`, `name`, `meter_name`, tenure dates, image, chamber, state).

### `Promise`

UUID `id`; `meter_id`; `promise_tracker_id`; `status` (`not_yet_rated` | `in_the_works` | `stalled` | `compromised` | `kept` | `broken`); `title`; `source`; `subject_area`; `promise_blog`; `priority` (`high` | `medium` | `low`); `promise_img`; `description`; `promise_start_date`; `promise_end_date`; `admin_id`; `update_count`; `last_updated_at`; `published_status` (`in_review` | `published` | `rejected`); timestamps.

### `Meter`

UUID `id`; `promise_tracker_id`; `name` (official's name); `state_id` (nullable); `chamber` (nullable); `meter_name`; `meter_type` (`promise_tracker` | `promise_ring`); `description`; `tenure_date_start`; `tenure_date_end`; `meter_img`; `admin_id`; timestamps.

## Endpoint details

### 1–3. Current meter by type

`GET /presidential` · `GET /state/{state}` · `GET /legislative/{chamber}`

- Presidential: the current meter is the tracker whose tenure covers today.
- State: matched by state name (case-normalised); falls back to the most recently created state meter.
- Legislative: `chamber` ∈ `upper-chamber` | `lower-chamber` (mapped to `Upper Chamber` / `Lower Chamber`); picks the most recent matching meter.
- Response 200: `MeterData`. If no meter exists, the same shape returns with zeroed counts and message `No current meter found`.
- Errors: 500 on unexpected failure.
- Evidence: `TrackedPromise::currentPromises` → `PromiseService::getCurrentPromises`; website `PresidentialTracker.jsx`, `StateTracker.jsx`, `LegislativeTracker.jsx`.

### 4–5. Promises for a specific meter

`GET /meter/{meterId}/presidential` · `GET /meter/{meterId}/legislative/{chamber}`

- Response 200: `MeterData` for the requested meter (published promises only).
- Errors: 500.
- Evidence: `TrackedPromise::getPromiseWithMeter` → `PromiseService::getPromiseWithMeter`; website tracker pages.

### 6. Promise detail

`GET /view/promise/{promiseId}`

- Response 200 `data`: the promise with `meter`, `tracker`, `admin`, and published `updates` where `update_type = update`. May be `null` if not found.
- Errors: 500.
- Evidence: `TrackedPromise::viewPromise` → `PromiseService::viewPromise`.

### 7. Latest promises

`GET /latest/promises/fetch`

- Response 200 `data`: array of three random published promises, each with `tracker` and `meter`.
- Errors: 500.
- Evidence: `PromiseService::latestPromises`; website `TrackedPromise.jsx`.

### 8. Promise Ring by state

`GET /all/promise-ring/fetch/{state}`

- Response 200 `data`: `{ total_promises, stalled, compromised, kept, broken, in_the_works, not_yet_rated, promises }`; promises include `tracker`, `meter`, `admin`.
- Errors: 404 `State not found` if the state name is unknown; 500 otherwise.
- Evidence: `PromiseService::promiseRing`; website `PromiseRing.jsx`, `homepage.jsx`.

### 9. Reminders by tracker type

`GET /promise/reminders/fetch/{type}`

- `type` ∈ `presidential` | `state` | `legislative` (matched against `updates.promise_tracker_type`).
- Response 200 `data`: published reminder updates (`update_type = reminder`, `is_published = true`), each with its `promise`.
- Errors: 500.
- Evidence: `PromiseService::allPromiseReminders`; website homepage and `PromiseReminder.jsx`.

### 10. Single reminder

`GET /view/promise/reminder/{updateId}`

- Response 200 `data`: the update with its `promise`.
- Errors: 404 if not found; 500 otherwise.
- Evidence: `PromiseService::viewPromiseReminder`. No current website call found — controller-defined.

### 11. Leaderboard

`GET /all/leaderboards/scores?type={all|this_week}`

- **Target contract:** `type` ∈ `all` | `this_week`. `this_week` filters `leader_boards.created_at` to the current week; any other value returns all-time.
- Response 200 `data`: `{ "leaderboards": [...] }` — rows keyed by learner (`user_id`, `name`, `email`, `attempt_number`) with `total_score` (quiz score plus extra scores); only active users. Exact row fields pending route verification.
- Known defect: the website currently sends `type=Week`, which the backend does not match — the weekly branch never triggers. The approved fix makes the website send `this_week`.
- Errors: 500.
- Evidence: `TrackedPromise::leaderBoards` → `PromiseService::leaderboard`; website `Leaderboard.jsx`.

### 12. Top students

`GET /leaderboard/top/student`

- Response 200 `data`: top three active students by `total_score` (latest attempt's quiz score plus extra scores), each with `user_id`, `name`, `email`, `attempt_number`, `total_score`, `completed_courses` (courses watched ≥ 90%).
- Errors: 500.
- Evidence: `Leaderboard::topStudent` → `StudentService::fetchTopStudents`; website homepage.

### 13. Totals at a glance

`GET /total/promise/tracked`

- Response 200 `data`: `{ total_promises, stalled, compromised, kept, broken, in_the_works, not_yet_rated, reminders_sent }` — published promises only.
- Errors: 500.
- Evidence: `PromiseService::getPromiseTracked`; website `Glance.jsx`.

### 14. Cohort text

`GET /get/cohort/text`

- Response 200 `data`: the first `cohort_texts` row, or `null`.
- Errors: 500.
- Evidence: `PromiseService::getCohortText`.

### 15. Promise Ring states

`GET /all/promise-ring/states`

- Response 200 `data`: states for the default country (`NG`), ordered by name, each with its `promise_ring` meters and `promises_count` (published only).
- Errors: 500.
- Evidence: `PromiseService::getPromiseRingStates`.

### 16. Subscribe to meter

`POST /subscribe/to/meter`

- Request: `{ "email": "…", "meter_id": "…" }` — both required; `email` must be a valid email.
- Behaviour: upserts `meter_subscribers` by `(email, meter_id)` (application-level; no DB unique constraint).
- Response 200 `data`: the echoed request body.
- Errors: 422 validation; 500 otherwise.
- Evidence: `MeterSubscribers::subscribe` → `MeterSubscriberService::subscribe`; website `PromisePercentageDisplay.jsx`.

### 17. Pledge count

`GET /pledges/count`

- Response 200 `data`: `{ "count": 12345 }`.
- Errors: none expected beyond transport failures.
- Evidence: `VoterPledgeController::count`; website `usePledgeCount.js`.

### 18. Create pledge

`POST /pledges`

- Request:

```json
{
  "full_name": "required, max 255",
  "phone_number": "required, unique, max 30",
  "email": "optional, valid email, unique when normalised",
  "state": "required, max 120",
  "lga": "required, max 120",
  "pvc_status": "yes | no | processing",
  "accepted_pledge": true,
  "wants_updates": false
}
```

- Normalisation: `full_name` is trimmed and whitespace-collapsed; `normalized_full_name` (lower-cased) and `normalized_email` (lower-cased, trimmed) are enforced unique alongside `phone_number`.
- Response 201 `data`: `{ "count": …, "pledge_id": "uuid" }`.
- Errors: 422 with field-keyed errors (`full_name`, `phone_number`, `email`, `pledge`); duplicate submissions report a friendly message.
- Evidence: `VoterPledgeController::store`; website `PledgeToVote.jsx`; `VoterPledgeTest`.

### 19. Nominate friends

`POST /pledges/{pledgeId}/nominations`

- Request: `{ "emails": ["a@example.com", "b@example.com"] }` — array of 1–5 emails, distinct (case-insensitive), each valid RFC email, max 255. Emails are trimmed and lower-cased before validation.
- Limit: at most 10 unique emails per pledge across all requests.
- Behaviour: new emails are stored and queued on the `mail-high` queue via `SendVoterPledgeNomination`; `queued_at`/`sent_at` track delivery.
- Response 202 `data`: `{ "accepted": 2, "duplicates": 0 }`.
- Errors: 422 (per-request max, invalid emails, 10-unique limit); 500 otherwise.
- Evidence: `VoterPledgeController::nominate` + `StoreVoterPledgeNominationsRequest`; website `PledgeThankYou.jsx`.

### 20. Partners (historical)

`GET /all/current/partners`

- Response 200 `data`: enabled partners — `{ id, partners_name, partners_logo }`, newest first.
- Status: **historical** — present in the website's committed history but absent from current working source. Kept for reference until retirement is confirmed.
- Errors: 500.
- Evidence: `PromiseService::getPartners`; commit `328d1f4`.
