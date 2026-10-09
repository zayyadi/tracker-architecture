# Onboarding

Accurate as of: 2026-09-11
Goal: a new developer can set up, run, and land a meaningful change using only these docs.

## First-day path

1. Read the [system overview](../overview.md) — what Promise Tracker is and who uses it
2. Read the [architecture](../architecture.md) — current state, target state, and gaps
3. Set up the [backend](backend-setup.md)
4. Set up the [website](website-setup.md)
5. Land your [first meaningful change](first-change.md)

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| PHP | **8.2+** | Laravel 11's locked framework requires it (local 8.1 is a known blocker) |
| Composer | 2.x | Backend dependencies |
| Node.js | 18+ | Vite 5 for both frontends |
| npm | 9+ | Package management |
| MySQL | 8.x | Local database (SQLite works for the test suite) |
| Git | any recent | Version control (backend history arrives with the monorepo) |

## Current blockers (be honest with yourself here)

| Blocker | Impact | Fix |
|---|---|---|
| Local PHP is 8.1 | `artisan` and tests cannot run at all | Install PHP 8.2 first |
| `routes/` is missing | Every page and API route 404s | Route reconstruction ([ADR-0003](../adr/0003-reconstruct-missing-route-files.md)) |
| Backend has no Git history | No branches, no rollback | Root monorepo ([ADR-0001](../adr/0001-adopt-root-monorepo.md)) |
| `.env.example` is stock Laravel | Missing project keys (Cloudinary, Google, Mailtrap, Pusher) | Hygiene pass ([ADR-0007](../adr/0007-scoped-hygiene-pass.md)) |

You can still complete setup and explore the codebase today; serving routes and running feature tests wait on the fixes above.

## Where to get help

- [API contracts](../api/README.md) · [Data model](../data/README.md) · [Deployment](../deployment/README.md) · [Security](../security/README.md) · [Testing](../testing/README.md)
- Decisions and their reasoning: [ADR index](../adr/)
