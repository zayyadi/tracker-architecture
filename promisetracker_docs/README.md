# Promise Tracker Documentation

> Cross-system target design agreed 2026-10-09: start with the [unified documentation](../README.md) and [architecture](../docs/architecture.md). This set remains supporting platform evidence; runtime claims require verification.

Accurate as of: 2026-09-10

This is the single documentation home for both parts of the platform:

| Canonical name | What it is |
|---|---|
| `promise_tracker` | The Laravel 11 backend, public API, learner services, and embedded Vue 3 admin console |
| `promise_tracker_website` | The React 18 public website (package name `advokc` — legacy) |

Promise Tracker is a project of the AdvoKC Foundation.

## Read this first

- [System overview](overview.md) — plain-language introduction for every audience
- [Architecture](architecture.md) — current state, target state, and gap analysis
- [Onboarding](onboarding/README.md) — set up, run, and land your first change
- [Decisions](adr/) — accepted architecture decisions and their reasoning

## Document map

| Area | Document | Status |
|---|---|---|
| Overview | [overview.md](overview.md) | Phase 1 — complete |
| Architecture | [architecture.md](architecture.md) | Phase 1 — complete |
| Data flows | [data-flow.md](data-flow.md) | Complete |
| API contracts | [api/](api/) | Phase 2 — complete |
| Data model | [data/](data/) | Phase 2 — complete |
| Deployment | [deployment/](deployment/) | Phase 3 — complete |
| Security | [security/](security/) | Phase 3 — complete |
| Testing | [testing/](testing/) | Phase 3 — complete |
| Accessibility & low-connectivity report | [testing/accessibility-low-connectivity-report.md](testing/accessibility-low-connectivity-report.md) | Complete |
| Onboarding | [onboarding/](onboarding/) | Phase 4 — complete |
| Decisions | [adr/](adr/) | Ongoing |

## Conventions

- All diagrams are [Mermaid](https://mermaid.js.org/) — no binary images.
- Every document states an **Accurate as of** date.
- Architecture decisions are recorded as lightweight, numbered ADRs in `adr/` (`0001-title.md`).
- Documentation is written in English.
- Gap priorities are ordered: security > correctness > operability > polish.
