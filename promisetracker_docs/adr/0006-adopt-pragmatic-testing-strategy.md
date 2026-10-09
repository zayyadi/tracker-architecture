# ADR-0006: Adopt the pragmatic testing strategy

- Status: Accepted
- Date: 2026-09-11

## Context

The backend has 9 PHPUnit test methods; every feature test targets routes that don't exist, so all of them fail until route reconstruction. The website has no tests at all. There is no CI. A comprehensive, coverage-driven strategy would be expensive and mostly unrunnable today.

## Decision

Adopt the strategy in [docs/testing/README.md](../testing/README.md):

- **API contract tests** for the public endpoints and admin auth (first priority)
- **Website smoke tests** for pledge submission and tracker reads (Playwright)
- **Data-integrity tests** for uniqueness and limit invariants
- **A minimal pipeline** after the monorepo: backend tests on PHP 8.2, both builds, website lint
- **No coverage thresholds** — the documented critical flows are the bar

## Ground rule

Reconstructed routes must make the existing feature tests pass before any other refactor.

## Consequences

- Effort concentrates on the flows that carry PII and public trust.
- Some code remains untested by policy; that is accepted and visible.
- Adding the website smoke suite introduces Playwright as a dev dependency.
