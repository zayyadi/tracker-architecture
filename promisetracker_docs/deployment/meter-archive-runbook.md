# Meter Archive — Live Server Runbook

Date: 2026-09-21
Scope: archive any meter (`promise_tracker` or `promise_ring`) with read-only freeze + email approval.
Example: archive Ekiti tenure-1 Promise Ring, create tenure-2 fresh, link them.

> All backend enforcement is in the API (403 on writes). Frontend badge/banner snippets below are
> optional courtesy UI — the system is safe without rebuilding frontends.

## 1. What this release does

- Adds `meters.is_archived`, `archived_at`, `archived_by`, `parent_meter_id`.
- Adds `meter_archive_requests` approval queue (`pending` → `approved` / `rejected`).
- Flow: any editor (`create-meter`) **requests** archive/unarchive → mail to `ARCHIVE_APPROVAL_EMAIL`
  → **only** that email's admin account can **approve/reject**. Meter stays editable until approval.
- On approve `archive`: meter becomes read-only (all writes 403). On approve `unarchive`: editable again.
- New tenure meters start empty; set `parent_meter_id` to the archived meter for
  "View current / View archive" navigation. No auto-cloning of promises.
- Archived meters are excluded from `current-meter` auto-pick (admin + public presidential/state/
  legislative + Promise Ring list) but remain readable directly by ID.
- `in_review` / unpublished items freeze as-is. Subscribers untouched.

## 2. Files changed / added

| # | File | Change |
|---|---|---|
| 1 | `database/migrations/2026_09_22_000001_add_archive_columns_to_meters_table.php` | NEW — archive columns + indexes |
| 2 | `database/migrations/2026_09_22_000002_create_meter_archive_requests_table.php` | NEW — approval queue table |
| 3 | `app/Models/Meter.php` | casts, `parent/children/archiveRequests`, `isArchived/isEditable`, `scopeActive/scopeArchived` |
| 4 | `app/Models/MeterArchiveRequest.php` | NEW model |
| 5 | `app/Traits/GuardsMeterArchive.php` | NEW — `archivedMeterError()` + `findMeter()` |
| 6 | `app/Mail/MeterArchiveMail.php` + `resources/views/emails/meter-archive.blade.php` | NEW approval/decision mail |
| 7 | `app/Jobs/SendMeterArchiveEmail.php` | NEW queued mail (`mail-high`) |
| 8 | `app/Services/Admin/Meter/MeterArchiveService.php` | NEW request/approve/reject/resend/pending |
| 9 | `app/Services/Admin/Meter/MeterService.php` | `parent_meter_id` on create; auto-pick excludes archived |
| 10 | `app/Services/Admin/Promise/PromiseService.php` | 403 guards on create/update/delete/import/publish/unpublish/updates |
| 11 | `app/Services/Api/Promise/PromiseService.php` | auto-pick excludes archived; Ring list = active only; selects expose `is_archived,parent_meter_id` |
| 12 | `app/Imports/PromisesImport.php` | skips rows for archived meters |
| 13 | `app/Http/Controllers/Admin/Promise/Promise.php` | `importPromises` pre-check 403 |
| 14 | `app/Http/Controllers/Admin/Meter/MeterArchive.php` | NEW controller |
| 15 | `database/seeders/PermissionSeeder.php` | adds `archive-meter`, `unarchive-meter` (fresh installs) |
| 16 | `database/seeders/MeterArchivePermissionSeeder.php` | NEW live-safe seeder (idempotent) |
| 17 | `.env.example` | documents `ARCHIVE_APPROVAL_EMAIL` |
| 18 | `routes_snippet/meter-archive-routes.php` | COPY-PASTE routes for live `routes/api.php` |

No composer dependencies added. No existing API paths changed.

## 3. Pre-flight (live)

```bash
# 1. Backup database (run on live, keep the file off-repo)
mysqldump -u promise_tracker_user -p promise_tracker_db > /root/backup_pre_archive_$(date +%F).sql

# 2. Confirm live PHP + queues
php -v            # expect 8.2+
sudo systemctl status promise-tracker-queue promise-tracker-mail-high-queue

# 3. Confirm approver account exists and its exact email
#    (must be a Super Administrator; mail goes to this address)
mysql -u promise_tracker_user -p -e "SELECT id,name,email FROM admins WHERE email='YOU-APPROVER@domain';" promise_tracker_db
```

Decide the value now: `ARCHIVE_APPROVAL_EMAIL` (e.g. the same address as `PROMISE_NOTIFICATION_EMAIL=habib@advokc.ng`, or a different owner). Only that inbox can approve.

## 4. Deploy steps (live, in order)

### 4.1 Sync code
Copy the files in §2 to `/var/www/promise_tracker/` preserving paths. Do NOT copy `.env`.

### 4.2 Set env
```bash
sudo nano /var/www/promise_tracker/.env
# add:
ARCHIVE_APPROVAL_EMAIL=habib@advokc.ng
```
Use the real approver inbox. Save, then:
```bash
php artisan config:clear
```

### 4.3 Migrate (safe, additive only)
```bash
cd /var/www/promise_tracker
php artisan migrate --force
# expect:
#   2026_09_22_000001_add_archive_columns_to_meters_table .... DONE
#   2026_09_22_000002_create_meter_archive_requests_table ... DONE
```
Verify:
```bash
php artisan migrate:status | grep -i archive
mysql -u promise_tracker_user -p -e "DESCRIBE meters; DESCRIBE meter_archive_requests;" promise_tracker_db
# meters must show is_archived (tinyint 0), archived_at, archived_by, parent_meter_id
# all existing rows: is_archived = 0 (nothing freezes on deploy)
```

### 4.4 Seed permissions (idempotent, safe to re-run)
```bash
php artisan db:seed --class=MeterArchivePermissionSeeder --force
```
Verify:
```bash
mysql -u promise_tracker_user -p -e "SELECT name FROM permissions WHERE name LIKE '%archive-meter%';" promise_tracker_db
# archive-meter, unarchive-meter
```

### 4.5 Add routes (live `routes/api.php` only — file is missing in this local copy)
Open live `routes/api.php`, find the meter group (`meter/create`, `meter/all`), paste inside it:

```php
Route::post('meter/request-archive/{meterId}', [MeterArchive::class, 'requestArchive'])
    ->middleware('permission-checker:create-meter');
Route::post('meter/resend-archive/{requestId}', [MeterArchive::class, 'resend'])
    ->middleware('permission-checker:create-meter');
Route::get('meter/archive-requests/pending', [MeterArchive::class, 'pending'])
    ->middleware('permission-checker:create-meter');
Route::post('meter/approve-archive/{requestId}', [MeterArchive::class, 'approve'])
    ->middleware('permission-checker:archive-meter|unarchive-meter');
Route::post('meter/reject-archive/{requestId}', [MeterArchive::class, 'reject'])
    ->middleware('permission-checker:archive-meter|unarchive-meter');
```
Add `use App\Http\Controllers\Admin\Meter\MeterArchive;` at the top. Full snippet also saved at
`routes_snippet/meter-archive-routes.php`.
```bash
php artisan route:clear && php artisan config:clear
php artisan route:list | grep -i archive
# expect 5 rows above
```

### 4.6 Restart workers + caches
```bash
php artisan queue:restart
sudo systemctl restart promise-tracker-queue promise-tracker-mail-high-queue
sudo systemctl status promise-tracker-queue promise-tracker-mail-high-queue
```

## 5. Verify (do this before announcing)

```bash
TOKEN=<admin-bearer-token>
BASE=https://admin.promisetracker.ng/api/v1/admin
METER=<ekiti-tenure-1-meter-id>

# 1. Request archive (as editor with create-meter)
curl -s -X POST "$BASE/meter/request-archive/$METER" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"archive","reason":"Tenure 1 ended"}'
# -> 200 "Awaiting approval." + request id. Meter STILL editable here.

# 2. Approver inbox receives mail (Mailtrap/live inbox). If not:
tail -f storage/logs/laravel.log | grep -i archive
php artisan queue:failed | head

# 3. Non-approver tries to approve -> 403
curl -s -X POST "$BASE/meter/approve-archive/<requestId>" \
  -H "Authorization: Bearer <other-admin-token>"
# -> 403 "Only <approver> can approve this request"

# 4. Approver approves -> 200, meter frozen
curl -s -X POST "$BASE/meter/approve-archive/<requestId>" \
  -H "Authorization: Bearer <approver-token>"

# 5. Writes now 403, reads still 200
curl -s -X POST "$BASE/promise/create" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"x","source":"s","description":"d","start_date":"2024-01-01","end_date":"2024-02-01","meter_id":"'$METER'","head_tracker_id":1,"promise_tracker_id":"<tracker>","promise_blog":"b","status":"not_yet_rated","priority":"high"}'
# -> 403 "Meter is archived and read-only"
curl -s "$BASE/promise/all/$METER" -H "Authorization: Bearer $TOKEN"  # -> 200
```

Ekiti walkthrough: request archive tenure-1 → approve → create tenure-2 via `meter/create`
with `"parent_meter_id": "<tenure-1-id>"` → confirm `meter/current-meter/null/<tracker>/promise_ring`
no longer returns tenure-1, and tenure-1 fetched by ID shows `is_archived: true`.

## 6. API contract (new endpoints)

| Method | Path | Perm | Body | Success |
|---|---|---|---|---|
| POST | `meter/request-archive/{meterId}` | `create-meter` | `{type: archive\|unarchive, reason?}` | 200 + request (pending). 409 if duplicate pending / already in target state |
| POST | `meter/approve-archive/{requestId}` | `archive-meter\|unarchive-meter` + email match | — | 200, meter flipped |
| POST | `meter/reject-archive/{requestId}` | same as approve | `{reason?}` | 200, meter unchanged |
| POST | `meter/resend-archive/{requestId}` | `create-meter` | — | 200 reminder mail |
| GET | `meter/archive-requests/pending` | `create-meter` | — | 200 list |

Blocked-write error (all promise/meter writes on archived): `403 {success:false, message:"Meter is archived and read-only"}`.

## 7. Optional frontend cues (no rebuild required for safety)

Backend already returns `is_archived` / `parent_meter_id` on meters. When you next build:

**Vue admin (`ViewPromiseTracker.vue` / `PromiseRing.vue`)**
```vue
<span v-if="currentTenure?.is_archived" class="archived-badge">Archived tenure — read-only</span>
<!-- disable writes: -->
<button :disabled="currentTenure?.is_archived" title="Archived tenure — read-only">Add promise</button>
```
Meter list: show `Archived` badge where `meter.is_archived`, add a "Show archived" filter (default: show all, archived never auto-selected as current).

**React website (`PromiseRing.jsx`, tracker pages)**
```jsx
{current_meter?.is_archived && (
  <div className="archive-banner">This is an archived tenure (read-only). <a href={`/ring/${current_meter.parent_meter_id ?? ''}`}>View current tenure</a></div>
)}
```

## 8. Rollback (if needed)

Data-safe: rollback only removes the new columns/table; existing promises untouched.
```bash
# 1. Remove the 5 pasted routes, then:
php artisan route:clear
# 2. Roll back just these two migrations (newest first):
php artisan migrate:rollback --step=2 --force
# 3. Remove env line ARCHIVE_APPROVAL_EMAIL, then config:clear + queue:restart
# 4. Optional: revoke permissions via admin UI (leave rows — harmless)
```
If approver email was wrong: just fix `.env`, `config:clear`, then `resend-archive/{requestId}` — no rollback needed.

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ARCHIVE_APPROVAL_EMAIL is not configured` | env missing/cached | set `.env`, `config:clear` |
| Approval mail never arrives | `mail-high` worker down / mail config | `systemctl status promise-tracker-mail-high-queue`; `php artisan queue:failed`; check Mailtrap/live SMTP |
| `Only X can approve` for the right person | email case/whitespace mismatch or wrong account | login as the exact inbox address; compare `admins.email` |
| `route:list` shows no archive routes | snippet pasted outside admin group / cache | re-paste inside group, `route:clear`, check `use` import |
| `permission` 403 on approve even as approver | role lacks `archive-meter` | run seeder §4.4 or grant via admin UI |
| Tenure-2 overlaps tenure-1 dates | state/governor overlap check is commented out (pre-existing) | allowed by design; link via `parent_meter_id`, do not rely on dates |
