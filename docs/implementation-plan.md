# Unified implementation plan

Design agreed: 2026-10-09. Status: planned application work; documentation completed in this workspace.

Use [architecture](architecture.md) and [channel contracts](channel-contracts.md) as target decisions. The older [RAG implementation plan](../ragdocs/implementation-plan.md) conflicts with newer BGE/worker documentation; inspect source before reusing its tasks.

## Phase 0 — Executable baseline

Owners: platform and RAG maintainers.

- [ ] Obtain Laravel, React, and Python sources; identify production revisions and Mai Gaskiya ownership.
- [ ] Record actual routes, widget/API contract, source schema, model configuration, and supervisors.
- [ ] Check documented missing routes, runtime mismatch, and credential gaps against current source; resolve verified blockers.
- [ ] Measure compressed transfer, cold/warm page timing, chat stages, source lag, and traffic/concurrency.
- [ ] Trace one promise/update from staff publication through sync, index, answer, and all channels.

Exit: runnable staging, safe configuration, known deployed behavior, and measured baseline.

## Phase 1 — Public data and grounded answers

Owners: platform data and RAG maintainers.

- [ ] Map public fields, source URLs, exact ratings, meter/tenure/archive state, and filters.
- [ ] Restrict extraction to approved published views; exclude private data.
- [ ] Verify/implement cursor, withdrawal handling, complete reconciliation, staged activation, and watermark.
- [ ] Enforce five-minute freshness, fallback policy, and cache invalidation.
- [ ] Verify hybrid retrieval/reranking; calibrate gate against adjudicated questions.
- [ ] Use complete filtered data for counts/rankings; prevent top-k population claims.

Exit: publication/rating/unpublication/deletion/archive fixtures pass; withdrawn evidence cannot survive caches/fallbacks; delayed current-status claims are suppressed.

## Phase 2 — Channels and durability

Owners: RAG and frontend maintainers.

- [ ] Add session/job API and existing-route compatibility adapter.
- [ ] Implement embedded and lightweight standalone clients using the shared renderer; defer embedded code until opened.
- [ ] Verify durable admission/outbox, supervision, deduplication, sender ordering, retries, and dead-letter handling.
- [ ] Configure WhatsApp and one Telegram ingress mode; verify real signed ingress and delivery.
- [ ] Enforce isolated conversations, job authorisation, pseudonymisation, and approved finite retention.
- [ ] Exercise reconnect, double send, API/worker restart, provider timeout, and index/model failure.

Exit: all four chat surfaces pass the same evidence fixtures and live channel smoke checks; accepted work survives restart and retries do not create duplicate answers.

## Phase 3 — Low-connectivity delivery

Owners: frontend and hosting maintainers.

- [ ] Apply image/font optimisation, route splitting, chat deferral, compression, and caching from [recommendations](low-connectivity.md).
- [ ] Compare bytes, appearance, content, accessibility, and timing after meaningful changes.
- [ ] Add bounded public-page offline cache and freshness labels; exclude private/chat responses.
- [ ] Investigate public-page prerendering only if measured rendering delay remains a bottleneck.

Exit: representative weak-network flows work, budgets are measured, and quality checks pass. Record any target miss with evidence.

## Phase 4 — Evaluation and staged release

Owners: editorial evaluator, maintainers, operations.

- [ ] Adjudicate cases covering names/regions, English/Pidgin usage, exact ratings, temporal/archive questions, ambiguous follow-ups, unsupported questions, withdrawals, and prompt injection.
- [ ] Compare baseline and candidate per slice on held-out cases; record retrieval coverage, citation validity, rating fidelity, and appropriate abstention. Provisional evaluator cases do not prove production answer quality.
- [ ] Block release on any incorrect official rating, withdrawn/private evidence disclosure, or cross-user leak in release fixtures; factual answers require valid citations.
- [ ] Verify source freshness, capacity, bounded queue age, provider recovery, monitoring alerts, backups, and restore.
- [ ] Release one browser client first, then the other browser surface and messaging cohorts; review correctness, latency, costs, and backlog before expanding.
- [ ] Record results and replace historical claims only after verification.

Rollback: retain compatible prior clients/API, pause failing channel admission, preserve durable jobs, and keep migrations backward compatible during rollout. Never restore an index containing withdrawn evidence. Use clear temporary status/abstention when fresh evidence is unavailable.

Completion requires source changes, compatibility checks, live channel configuration, evaluation, network measurements, and recovery evidence. This documentation workspace does not demonstrate those application outcomes.
