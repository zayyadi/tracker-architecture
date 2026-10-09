# Backend Setup — `promise_tracker`

Accurate as of: 2026-09-11

## What you are setting up

The Laravel 11 backend: public API, learner services, and the embedded Vue 3 admin console, backed by MySQL with database queues.

## Steps

### 1. Install dependencies

```bash
cd promise_tracker
composer install
npm install
```

### 2. Configure the environment

```bash
cp .env.example .env
php artisan key:generate
```

The stock `.env.example` defaults to SQLite and `log` mail. For a full local setup, also set the project keys listed in the [deployment configuration table](../deployment/README.md#configuration):

- `DB_CONNECTION=mysql` plus `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`
- `DEFAULT_COUNTRY`, `LEARNING_URL`, `PROMISE_NOTIFICATION_EMAIL`
- `CLOUDINARY_URL`, `CLOUDINARY_UPLOAD_PRESET`, `CLOUDINARY_NOTIFICATION_URL`
- `MAIL_MAILER` and Mailtrap settings
- `PUSHER_*` if you exercise broadcasting
- The Google service-account path (env-configured after the hygiene pass)

**Never copy secret values from the real `.env` in the tree** — it contains live credentials. Ask the maintainer for local/test values.

### 3. Migrate and seed

```bash
php artisan migrate --seed
```

`DatabaseSeeder` runs `AdminSeeder`, `PermissionSeeder`, and `AssignRoleToUser`, creating a local super admin:

- Email: `admin@tracker.co`
- Password: `password`

These are **local-development defaults only** — change them before any shared deployment.

### 4. Build the admin SPA

```bash
npm run build
```

### 5. Serve the app

```bash
php artisan serve
```

The app serves at `http://127.0.0.1:8000`. **Expected today: routes 404** until the route files are reconstructed ([ADR-0003](../adr/0003-reconstruct-missing-route-files.md)).

### 6. Run queue workers (optional, for jobs)

```bash
php artisan queue:work database --queue=default
php artisan queue:work database --queue=mail-high
```

### 7. Run the tests

```bash
DB_CONNECTION=sqlite DB_DATABASE=:memory: php artisan test
```

Requires PHP 8.2. The feature tests target routes that don't exist yet — they are the acceptance criteria for route reconstruction and will fail until then.

## Verify your setup

| Check | Expected today |
|---|---|
| `php artisan --version` | Laravel 11 on PHP 8.2 |
| `php artisan migrate:status` | All migrations ran |
| `npm run build` | Vite build succeeds |
| `php artisan serve` + `GET /up` | Health route responds |
| Any other route | 404 — blocked on missing routes |
