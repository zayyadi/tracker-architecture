# Admin API Contract

Accurate as of: 2026-09-10
Status: **Inferred** — 129 unique call patterns extracted from the embedded Vue admin SPA. Paths are relative to the backend origin; the SPA also calls `GET /sanctum/csrf-cookie` during bootstrap.

## Authentication

| Method | Path | Contract |
|---|---|---|
| POST | `api/v1/admin/authenticate` | Request `{ email, password }`. 200: envelope with `data = { user, access_token }` (Sanctum plain-text token, `admin` guard). 401: invalid credentials. 422: validation. |
| POST | `api/v1/admin/validate-token` | Returns `{ valid: true, user }` for a valid bearer token; 401 `{ valid: false, user: null }` otherwise. |
| GET | `api/v1/admin/logout` | Deletes the admin's tokens. 201 with the user envelope. |
| GET | `api/v1/admin/admin-management/my-profile` | Current admin profile. |

All other endpoints expect `Authorization: Bearer <token>` (see `promise_tracker/resources/js/auth.js`). Most return the shared envelope (see [README](README.md)).

## Endpoint inventory by module

### Admin & RBAC management

| Method | Path |
|---|---|
| GET | `admin-management/fetch-roles` |
| GET | `admin-management/fetch-admins` |
| GET | `admin-management/admin-user/{adminId}` |
| GET | `admin-management/role/{roleId}` |
| GET | `admin-management/get-logs` / `get-logs/` (filter params) |
| GET | `admin-management/get-user-logs/{adminId}` |
| GET | `admin-management/activity-logs/export` |
| GET | `admin-management/user-activity-logs/export/{adminId}` |
| GET | `admin-management/assign-permission/{roleId}/{permissionId}` |
| GET | `admin-management/revoke-permission/{roleId}/{permissionId}` |
| GET | `admin-management/regenerate-password/{userId}` |
| POST | `admin-management/create-role` |
| POST | `admin-management/create-admin` |
| POST | `admin-management/change-password` |
| POST | `admin-management/add-permission` |

### Dashboard

| Method | Path |
|---|---|
| GET | `dashboard/stats/{meter}` |
| GET | `dashboard/meters/{value}` |
| GET | `dashboard/get-governors/{state}` |
| GET | `dashboard/get-meter/{value}` |
| GET | `dashboard/leaderboard/top/student` |
| GET | `dashboard/notifications` |

### Promise trackers

| Method | Path |
|---|---|
| GET | `promise-tracker/all` |
| GET | `promise-tracker/get/{id}` |
| GET | `promise-tracker/get/tracker/type` |
| GET | `promise-tracker/get-reviews/{type}/{id}` |
| GET | `promise-tracker/confirm-read/{updateId}` |
| POST | `promise-tracker/create` |
| POST | `promise-tracker/reject-review/{id}/{type}` |
| PUT | `promise-tracker/update/{trackerId}` |
| DELETE | `promise-tracker/delete/{trackerId}` |

### Promises

| Method | Path |
|---|---|
| GET | `promise/all/{tenureId}` (optional `?page=`) |
| GET | `promise/all/promises/stats/` |
| GET | `promise/stats/{tenureId}` |
| GET | `promise/get/{id}` |
| GET | `promise/get-promise-update/{id}` |
| GET | `promise/get-promises-update/{id}` |
| GET | `promise/publish/{id}/{type}` and `promise/publish/{promiseId}` |
| GET | `promise/un-publish/{id}/{type}` and `promise/un-publish/{promiseId}` |
| POST | `promise/create` |
| POST | `promise/update/{promiseId}` |
| POST | `promise/create-promises-update/{promiseId}` |
| POST | `promise/status-promises-update/{promiseId}` |
| POST | `promise/update-promise-update/{promiseUpdateId}` |
| POST | `promise/import/promise/excel-file/{tenureId}/{trackerId}` |
| DELETE | `promise/delete/{promiseId}` |
| DELETE | `promise/delete-promise-update/{updateId}` |

### Meters, states & legislators

| Method | Path |
|---|---|
| GET | `meter/all` |
| GET | `meter/current-meter/{meterId}/{trackerId}` (optional `/{trackerType}`) |
| POST | `meter/create` |
| GET | `states/all` |
| GET | `states/get-governors/{stateId}` (optional `/{trackerType}`) |
| GET | `states/current-governor-meter/{governor}/{trackerId}` |
| GET | `legislators/get-assemblies/{chamber}` |
| GET | `legislators/current-legislative-meter/{assembly}/{trackerId}` |

### Cohorts

| Method | Path |
|---|---|
| GET | `cohorts/all` |
| GET | `cohorts/{cohortId}` |
| GET | `cohorts/students/{cohortId}?page=` |
| GET | `cohorts/student-in-temp-cohort/{cohortId}` |
| GET | `cohorts/end-cohort/{cohortId}` |
| GET | `cohorts/send-invitation/{cohortId}` |
| GET | `cohorts/get/countries` |
| GET | `cohorts/get/country/state/{countryId}` |
| POST | `cohorts/create` |
| POST | `cohorts/add/student` |
| POST | `cohorts/migrate-students/{cohortId}` |
| POST | `cohorts/import/student/excel-file/{cohortId}` |

### Students

| Method | Path |
|---|---|
| GET | `students/all` |
| GET | `students/view/{studentId}` |
| GET | `students/extra-points/{studentId}` |
| GET | `students/activate-students/{studentId}` |
| GET | `students/deactivate-students/{studentId}` |
| POST | `students/add-points/` |
| DELETE | `students/remove-extra-point/{pointId}` |

### Instructors

| Method | Path |
|---|---|
| GET | `instructors/all?page=` |
| GET | `instructors/view-instructor/{instructorId}` |
| POST | `instructors/create-instructor` |

### Resource management (courses)

| Method | Path |
|---|---|
| GET | `resource-management/all-courses?page=` |
| GET | `resource-management/course/{courseId}` |
| GET | `resource-management/instructors` |
| GET | `resource-management/archive-course/{courseId}` |
| GET | `resource-management/remove-archive-course/{courseId}` |
| GET | `resource-management/completion-rate/{courseId}` |
| POST | `resource-management/add-course` |
| POST | `resource-management/update-course/{courseId}` |
| POST | `resource-management/upload` |
| POST | `resource-management/upload-complete` |
| POST | `resource-management/` (dynamic suffix — ambiguous call site) |
| DELETE | `resource-management/delete/{courseId}` |
| DELETE | `resource-management/student-delete/{studentId}` |

### Course resources & quizzes

| Method | Path |
|---|---|
| POST | `resource/upload-resource/{courseId}` |
| DELETE | `resource/delete-resource/{resourceId}` |
| GET | `quiz/get-quiz/{courseId}` |
| POST | `quiz/add-quiz/{courseId}` |

### Analytics

| Method | Path |
|---|---|
| GET | `analytics/analytics-statistics` |
| GET | `analytics/get-cohorts` |
| GET | `analytics/learning-analytics/{cohortId}` |
| GET | `analytics/compare-cohorts/{cohortOne}/{cohortTwo}` |
| GET | `analytics/meter_type/{meterType}/{promiseType}` |
| GET | `analytics/meter_type/{value}` |
| GET | `analytics/promise/{meterType}/{meterId}/` |

### Partnerships, notifications, pledges & misc

| Method | Path |
|---|---|
| GET | `partnerships/all` · `partnerships/enable/` · `partnerships/disable/` |
| POST | `partnerships/create` |
| GET | `notifications/{type}?start_date=&end_date=&page=` |
| DELETE | `notifications/delete/{notificationId}` |
| DELETE | `notifications/delete/{startDate}/{endDate}/{type}` |
| GET | `pledge-dashboard/overview` · `by-state` · `by-lga` · `heatmap` · `reminder-contacts` |
| POST | `pledge-dashboard/send-reminder` |
| GET | `cohort-text/` |
| POST | `cohort-text/update` |
| GET | `user/all` |

## Contract observations

1. **REST violations are common** — state changes use GET (`publish`, `activate`, `assign-permission`), and several paths carry trailing slashes. Reconstructing routes is the moment to normalise naming, but changing method semantics would break the SPA; do both together or neither.
2. **Duplicate paths exist** (`promise/publish/{id}/{type}` and `promise/publish/{promiseId}`) — route ordering and parameter constraints matter.
3. **One ambiguous call site** posts to `resource-management/` with a dynamic suffix; it needs clarification during route reconstruction.
