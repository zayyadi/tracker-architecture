# ADR-0008: Correct the leaderboard contract

- Status: Accepted
- Date: 2026-09-11

## Context

The website sends `?type=Week` for the weekly leaderboard, but the backend checks `this_week` (`PromiseService::leaderboard`). The weekly branch can never match, so "weekly" silently behaves as all-time.

## Decision

The contract is `type=all|this_week`:

- the backend keeps `this_week` as the weekly selector;
- the website sends `this_week`;
- the public API contract documents both values.

## Consequences

- The weekly leaderboard works as intended.
- Any client still sending `Week` receives all-time results (the backend treats any non-`this_week` value as unfiltered) — backward-compatible but misleading, so the website fix ships with the contract doc.
- The testing strategy adds an explicit `all` vs `this_week` contract test.
