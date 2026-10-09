# ADR-0005: Markdown-first API contracts; defer OpenAPI

- Status: Accepted
- Date: 2026-09-11

## Context

The route files are missing, so no machine-readable route truth exists. The public, admin, and learner surfaces were reconstructed by hand from clients, controllers, services, Form Requests, and tests. OpenAPI would normally be generated or validated against real routes.

## Decision

The Markdown tables in `docs/api/` are the source of truth for API contracts at this stage. OpenAPI is deferred.

## Deferred-trigger review

Reopen this decision when all of the following hold:

- routes are reconstructed and verified ([ADR-0003](0003-reconstruct-missing-route-files.md));
- more than one consumer needs a machine-readable spec, or CI wants contract testing;
- the endpoint inventory has stabilised after the first verification pass.

## Consequences

- Contracts are human-readable, diffable, and fast to maintain today.
- No code generation, spec validation, or contract testing until OpenAPI arrives.
- The Markdown docs must be updated in the same change as any contract change.
