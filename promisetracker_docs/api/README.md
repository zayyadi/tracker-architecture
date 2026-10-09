# Promise Tracker — API Contracts

Accurate as of: 2026-09-10
Status: **Inferred** — pending route reconstruction and PHP 8.2 test verification ([ADR-0003](../adr/0003-reconstruct-missing-route-files.md))

## Inference discipline

The backend's route files are missing. Every contract below is inferred from the strongest available local evidence:

- HTTP calls made by the React public website (`promise_tracker_website/src`)
- HTTP calls made by the embedded Vue admin SPA (`promise_tracker/resources/js`)
- Laravel controllers, services, and Form Requests (`promise_tracker/app`)
- PHPUnit expectations (`promise_tracker/tests`)

Paths, middleware, route names, and ordering remain unverified until routes are reconstructed. When that happens, each document's status moves from **Inferred** to **Verified**.

## Surfaces

| Surface | Base | Auth | Document |
|---|---|---|---|
| Public API | `https://admin.promisetracker.ng/api/v1/promise-tracker` | None (public reads and writes) | [public.md](public.md) |
| Admin API | `api/v1/admin` (relative to the backend origin) | Sanctum bearer token, `admin` guard | [admin.md](admin.md) |
| Learner API | Unknown prefix | Sanctum bearer token, learner guard | [learner.md](learner.md) |
| Chatbot | `https://chat.promisetracker.ng/chat/web` | External service, not part of this API | Not documented here |

## Response envelope

Most endpoints share one envelope, produced by `App\Http\Controllers\Controller::sendResponse` / `sendError`:

```json
{
  "success": true,
  "data": {},
  "message": "…"
}
```

```json
{
  "success": false,
  "message": "…",
  "errors": {}
}
```

Known exceptions: `validate-token` endpoints return `{ "valid": bool, "user": … }` directly.

## Status codes in use

| Code | Meaning |
|---|---|
| 200 | Success |
| 201 | Resource created (pledge, logout token deletion) |
| 202 | Accepted for queueing (nomination emails) |
| 401 | Unauthenticated / invalid credentials / invalid token |
| 404 | Not found (state, reminder) |
| 422 | Validation error, with field-keyed `errors` |
| 500 | Unexpected server error |

## Versioning

All known backend routes live under `/api/v1`. The public website consumes `/api/v1/promise-tracker/...`; the admin SPA consumes `/api/v1/admin/...`.
