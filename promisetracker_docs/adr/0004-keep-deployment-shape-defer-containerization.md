# ADR-0004: Keep the deployment shape; defer containerisation

- Status: Accepted
- Date: 2026-09-10

## Context

The platform runs on Linux with Apache in front of Laravel's `public/index.php`, MySQL for data, and a systemd unit running the database queue worker as `www-data` from `/var/www/promise_tracker`. This shape works in production. Laravel Sail is present as a dependency, but there is no current CI requirement or second environment that demands containers.

## Decision

Keep the current deployment shape as the v1 target: Linux + Apache + systemd + MySQL. Document it properly and fix configuration drift (`APP_ENV` vs production URL, broadcast variable mismatch, `CLOUDINARY_URL` prefix). Defer containerisation.

## Deferred-trigger review

Reopen this decision if any of the following occur:

- a reproducible second environment (staging or CI) becomes necessary;
- CI tooling requires containers;
- environment-drift incidents recur after the de-drift work.

## Consequences

- The fastest path to an operable, documented environment.
- No container orchestration skills are required to run the platform.
- A future container migration remains a documented option rather than a forgotten idea.
