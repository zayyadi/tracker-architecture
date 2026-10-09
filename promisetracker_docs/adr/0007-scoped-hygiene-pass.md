# ADR-0007: Approve the scoped hygiene pass

- Status: Accepted
- Date: 2026-09-11

## Context

The security review found credentials in plain files, build artifacts and SQL dumps in the trees, config drift, and a PHP constraint mismatch. The project is documentation-first, but baselining the backend into Git without hygiene would commit secrets to history.

## Decision

Approve a tightly-scoped hygiene pass, and nothing more:

1. `.gitignore` covering `.env`, `vendor/`, `dist/`, `public/build`, SQL dumps, and service-account JSONs
2. Extend the stock `.env.example` with the project's real configuration keys (no live values)
3. Load the Google service-account via an env-configured path instead of a repo file
4. Fix `composer.json` PHP constraint: `^8.1` → `^8.2`
5. Align the broadcast variable (`BROADCAST_DRIVER` vs `BROADCAST_CONNECTION`)
6. Fix the doubled `CLOUDINARY_URL=` prefix

**Nothing is deleted.** Secrets and dumps stay on disk, ignored.

## Explicitly out of scope

- Rotating exposed credentials (operational follow-up)
- Git history sanitisation (separate follow-up)
- Any functional or refactoring change beyond the six items

## Consequences

- The tree becomes safe to baseline into the root monorepo ([ADR-0001](0001-adopt-root-monorepo.md)).
- Production `.env` must receive the same config corrections in parallel.
- The hygiene pass is a prerequisite for the monorepo, not a competitor to it.
