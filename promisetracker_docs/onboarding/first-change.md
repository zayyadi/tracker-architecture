# Your First Meaningful Change

Accurate as of: 2026-09-11

## Worked example: fix the leaderboard parameter

This is the approved correctness fix from [ADR-0008](../adr/0008-correct-leaderboard-contract.md) — small, real, and fully documented.

### 1. Read the contract

[Public API — Leaderboard](../api/public.md) defines the target contract: `type=all|this_week`. The backend checks `this_week`; the website currently sends `Week`, so the weekly branch never triggers.

### 2. Locate the call

`promise_tracker_website/src/pages/Leaderboard.jsx` builds the request:

```js
const response = await fetch(`${API_BASE_URL}/all/leaderboards/scores?type=${param}`);
```

### 3. Make the change

Change the parameter value sent for the weekly view from `Week` to `this_week`. Do not change the backend — it already implements the target contract.

### 4. Verify

```bash
cd promise_tracker_website
npm run lint
npm run build
```

Optionally run the dev site against a local backend and confirm the weekly leaderboard request URL contains `type=this_week`.

### 5. Commit

Follow the monorepo workflow ([ADR-0001](../adr/0001-adopt-root-monorepo.md)): branch from `main`, keep the change small, run the checks above, open a pull request. Until the monorepo lands, only the website has Git history — coordinate with the maintainer for backend changes.

## The general workflow

1. **Read first** — the relevant contract or data doc, then the code
2. **Branch** — one change per branch, named for the intent
3. **Check** — run the narrowest relevant check (lint, build, or the PHPUnit filter), then the broader suite before handing off
4. **Document** — if the change alters a contract, schema, or decision, update the matching doc and ADR in the same change
5. **Hand off** — small PR, clear description, link the doc you updated
