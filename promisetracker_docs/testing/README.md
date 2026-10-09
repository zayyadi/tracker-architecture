# Promise Tracker — Testing Strategy

Accurate as of: 2026-09-11
Approach: pragmatic (agreed) — inventory the current suites, define a critical-flow target, and avoid vanity coverage numbers.

## Current inventory

### Backend (`promise_tracker`)

PHPUnit with `Unit` and `Feature` suites (`phpunit.xml`). The testing environment uses array cache/session/mail, a sync queue, and reduced bcrypt rounds. SQLite in-memory is commented out in `phpunit.xml`, so it must be passed on the command line.

| Test file | Methods | What it covers |
|---|---|---|
| `tests/Feature/AdminAuthenticationTest.php` | 2 | Admin login rejects unknown email with 401; valid credentials return user + Sanctum token |
| `tests/Feature/VoterPledgeTest.php` | 5 | Unique pledge creation; normalised duplicate name rejected; case-insensitive duplicate email rejected; multiple pledges without email allowed; duplicate submission preserves the original |
| `tests/Feature/ExampleTest.php` | 1 | `GET /` returns 200 |
| `tests/Unit/ExampleTest.php` | 1 | Skeleton assertion only |

**Total: 9 test methods.** The feature tests target `/api/v1/admin/authenticate`, `/api/v1/promise-tracker/pledges`, and `/` — routes that do not exist in this copy. They therefore define the acceptance criteria for route reconstruction ([ADR-0003](../adr/0003-reconstruct-missing-route-files.md)) and will fail until the routes exist.

### Website (`promise_tracker_website`)

- `npm run lint` — ESLint with zero-warning policy
- No test runner, no test script, no test files

## How to run today

```bash
# Backend (requires PHP 8.2; local machine has 8.1 and is currently blocked)
cd promise_tracker
DB_CONNECTION=sqlite DB_DATABASE=:memory: php artisan test

# Website
cd promise_tracker_website
npm run lint
npm run build
```

## Gaps

1. All backend feature tests are blocked by the missing route files
2. Local PHP 8.1 cannot run the suite at all (vendor platform check requires 8.2)
3. No website tests despite public PII-submission flows
4. No CI pipelines exist at either app root
5. No API contract tests, no data-integrity tests, no queue-job tests

## Target strategy

The accessibility and low-connectivity baseline for the public website is captured in [accessibility-low-connectivity-report.md](accessibility-low-connectivity-report.md); its prioritised fixes are prerequisites for any "low-connectivity ready" claim.

### 1. API contract tests (backend, first priority)

Cover the public contract documented in [api/public.md](../api/public.md):

- Tracker reads: `/presidential`, `/state/{state}`, `/legislative/{chamber}` return the meter payload and status counts
- Pledge flow: create, all duplicate rejections, nullable-email behaviour
- Nominations: per-request max (5), per-pledge max (10 unique), queued state transitions
- Meter subscription: valid upsert + validation rejection
- Leaderboard: `type=all` vs `type=this_week` (the corrected contract)
- Admin auth: keep and extend the existing two tests

### 2. Website smoke tests

Two critical user journeys, run against a test backend or mocked API:

- **Pledge submission** — happy path, validation errors, duplicate handling
- **Tracker reads** — presidential/state/legislative pages render meter data from the API

A lightweight Playwright smoke suite fits both without a heavy unit-testing migration.

### 3. Data-integrity tests

- Voter pledge identity uniqueness (phone, normalised name, normalised email)
- Nomination limits and `queued_at`/`sent_at` transitions
- The future `meter_subscribers` unique constraint

### 4. Pipeline (after the monorepo)

Per change: backend tests on PHP 8.2, both apps' builds, website lint. No coverage thresholds — these flows are the bar.

## Ground rule

The reconstructed routes must make the existing feature tests pass **before** any other refactor. Tests first, refactors second.
