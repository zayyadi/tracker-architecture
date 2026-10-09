# Promise Tracker — Data Model

Accurate as of: 2026-09-10
Status: Derived from the 53 migrations in `promise_tracker/database/migrations`.

## Domain map

| Domain | Tables | Document |
|---|---|---|
| Promise tracking | `promise_trackers`, `meters`, `promises`, `updates` | [promise-tracking.md](promise-tracking.md) |
| Voter engagement (PII) | `voter_pledges`, `voter_pledge_nominations`, `pledge_reminder_contacts`, `meter_subscribers` | [voter-engagement.md](voter-engagement.md) |
| Identity & RBAC | `admins`, `users`, Spatie permission tables, `personal_access_tokens`, `password_reset_tokens`, `sessions` | [identity-rbac.md](identity-rbac.md) |
| Learning (LMS) | `modules`, `courses`, `instructors`, `questions`, `choices`, `answers`, `user_answers`, `leader_boards`, `extra_scores`, `course_progress`, `resources`, `cohorts`, `students`, `cohort_texts`, `student_migration_logs` | [learning.md](learning.md) |
| Media & operations | `media`, `transcription_trackings`, `messages`, `activity_logs`, `partners`, `jobs`, `failed_jobs`, `job_batches`, `cache`, `cache_locks` | [media-ops.md](media-ops.md) |
| Geography | `countries`, `states` | Covered in [promise-tracking.md](promise-tracking.md) |

## Data classification

| Class | Tables | Handling |
|---|---|---|
| Public content | `promise_trackers`, `meters`, published `promises`/`updates`, enabled `partners`, `cohort_texts`, `countries`, `states` | Public API exposes published/enabled rows only |
| Voter PII — highest sensitivity | `voter_pledges`, `pledge_reminder_contacts`, `voter_pledge_nominations` | Full names, phone numbers, emails, state/LGA, PVC status, consent flags. Never expose via public reads. |
| Learner PII | `users`, `students` | Names, emails, phone numbers, images, addresses, dates of birth |
| Credentials | `admins.password`, `users.password`, `personal_access_tokens`, `sessions`, `password_reset_tokens` | Hashed passwords and bearer tokens; treat as secrets |
| Operational | `activity_logs`, `messages`, `media`, `transcription_trackings`, `meter_subscribers`, job/cache tables | Internal; `meter_subscribers.email` is contact PII |

## Integrity findings

1. **Most relationships have no database foreign key.** Only four domain foreign keys are enforced: `leader_boards.user_id → users.id`, `questions.course_id → courses.id`, `user_answers.user_id → users.id` (all cascade), and `voter_pledge_nominations.voter_pledge_id → voter_pledges.id` (cascade). Spatie's permission pivots are also enforced. Every other relationship — promises→meters, meters→trackers, updates→promises, courses→instructors, and more — is convention-only, so orphaned rows are possible.
2. **`meter_subscribers` has no unique constraint on `(email, meter_id)`.** Deduplication happens only through `updateOrCreate` in the service layer.
3. **Mixed primary-key types.** Domain tables use UUIDs; `users`, `admins`, and Sanctum tokens use bigints. Cross-type joins are by convention and must stay deliberate.
4. **Soft deletes are widespread** (promises, meters, trackers, courses, questions, and others). Public queries must exclude both soft-deleted and unpublished rows.
5. **Voter pledges have strong identity guards**: unique `phone_number`, `normalized_full_name`, and `normalized_email`.
6. **Two student concepts exist**: `students` (pre-enrolment roster/invite list) and `users` (enrolled learners). Treat them as distinct.
