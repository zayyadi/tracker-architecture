# Website Setup — `promise_tracker_website`

Accurate as of: 2026-09-11

## What you are setting up

The React 18 public website: trackers, Promise Ring, pledges, leaderboard, and the chatbot widget. It is a Vite SPA with no backend of its own — it calls the Promise Tracker API.

## Steps

### 1. Install dependencies

```bash
cd promise_tracker_website
npm install
```

### 2. Configure the API base

Create `.env` in the app root:

```bash
VITE_API_BASE_URL=http://127.0.0.1:8000/api/v1/promise-tracker
VITE_CHATBOT_API_URL=https://chat.promisetracker.ng/chat/web
```

Point `VITE_API_BASE_URL` at your local backend, or keep the production URL for read-only browsing. **Warning:** pledge submissions and subscriptions are writes — they go wherever this variable points. Do not submit test pledges against production.

### 3. Run the dev server

```bash
npm run dev
```

### 4. Check and build

```bash
npm run lint
npm run build
npm run preview
```

There is no test suite yet — lint and a successful build are the current checks (see the [testing strategy](../testing/README.md)).
