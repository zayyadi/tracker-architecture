# ADR-0002: Stabilize the modular monolith

- Status: Accepted
- Date: 2026-09-10

## Context

`promise_tracker` is a Laravel 11 modular monolith: HTTP controllers, a service layer, Eloquent models, queue jobs, learner services, and an embedded Vue 3 admin console, backed by MySQL. The React public website consumes its API. There is no evidence of scaling pain that would justify splitting the system, and the team is small.

## Decision

Keep the single Laravel deployment as the target. Stabilise it by:

- formalising module boundaries (promise tracking, voter engagement, learning, identity/access, media, communications/ops);
- making the public API contract-first and documented;
- keeping the Vue admin SPA embedded;
- keeping the React website as the only public client.

## Rejected alternatives

| Alternative | Why rejected | Reopening trigger |
|---|---|---|
| Extract the public API into a dedicated service | No evidence of divergent scaling; adds deployment and operational cost | Sustained load where API scaling needs diverge from admin; independent deploy cadence; hard security-isolation requirement |
| Split into microservices | Small team, single product, no demonstrated bottleneck | Separate product lines with separate teams; proven per-domain scaling bottlenecks |

## Consequences

- Lower operational cost and one deployment to reason about.
- Module boundaries are enforced by convention, documentation, and tests rather than network boundaries.
- Extraction remains possible later because the modules are named and documented.
