# Learner API Contract

Accurate as of: 2026-09-10
Status: **Inferred — lowest confidence.** Controllers and services exist, but no client inside this repository calls them and the route files are missing. The URL prefix and exact paths are unknown; this surface may serve a mobile app or a future learner portal.

## Surface

Controllers live under `App\Http\Controllers\Learner\*` with services under `App\Services\Learner\*`. Authentication uses Sanctum tokens (learner identity in `users`).

| Group | Controller methods (inferred actions) |
|---|---|
| Auth | `authenticateLeaner`, `verifyOtp`, `forgotPassword`, `resetPassword`, `logOut` |
| User | `validateToken`, `learnerDetails`, `learnerCountries`, `learnerState/{countryId}`, `learnerUpdateProfile`, `learnerUpdateProfileImage` |
| Dashboard | `dashboard` |
| Courses | `courses`, `recentlyWatched`, `viewCourse/{courseId}`, `recordProgress`, `downloadResource/{resourceId}` |
| Quiz | `courseQuiz/{courseId}`, `submitQuiz` |
| Leaderboard | `leaderboard` (default limit 30) |

## What is known

- Responses use the shared envelope (see [README](README.md)) except `validateToken`, which returns `{ valid, user }`.
- Learners are stored in `users` (see [identity & RBAC data model](../data/identity-rbac.md)); quiz progress feeds `leader_boards` and `extra_scores`.

## Open questions

1. Which client consumes this surface (mobile app, another frontend, none)?
2. What is the URL prefix (`/api/v1/learner/...`)?
3. Are these routes deployed in production?

These questions must be answered before the route reconstruction in [ADR-0003](../adr/0003-reconstruct-missing-route-files.md) can cover the learner surface confidently.
