# ADR-0003: Reconstruct the missing route files

- Status: Accepted
- Date: 2026-09-10

## Context

`promise_tracker/bootstrap/app.php` expects `routes/web.php`, `routes/api.php`, and `routes/console.php`. None of them exist in the working copy, and they cannot be recovered from local Git history or route caches — the backend has no usable Git history. The website's calls and the backend's tests imply 23 public/admin endpoints, and controllers, services, and Form Requests define their shapes.

## Decision

Reconstruct the route files from the best available local evidence:

- the 23-endpoint inventory inferred from the website and tests;
- controller and service method signatures;
- Form Request validation rules;
- PHPUnit expectations.

The reconstruction is explicitly marked as inferred, recorded in this ADR, and verified with the test suite once PHP 8.2 is available. If the original route files are later recovered from production or another copy, they supersede the reconstruction.

## Consequences

- The backend becomes runnable and its routing verifiable locally.
- Inferred middleware, ordering, route names, and parameter constraints may differ from production until originals are reconciled.
- The API contract documentation gains a code-backed source of truth instead of describing a system that cannot route.
