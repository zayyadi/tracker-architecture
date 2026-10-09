# Meter Archive — All Code Changes (step-by-step)

Date: 2026-09-21
Companion runbook: `meter-archive-runbook.md` (deploy order, verify curl, rollback).
This file is the complete build log: every file created or edited, in the order to apply on live, with exact code.

Spec locked (grill Q1–Q18): unified flag for `promise_tracker` + `promise_ring`; full write-freeze (403) with reads intact; reversible via request → mail to `ARCHIVE_APPROVAL_EMAIL` → only that inbox approves; fresh second tenure linked by `parent_meter_id`; excluded from current-pick + badge/banner; manual only; pending `in_review` frozen as-is; subscribers untouched.

Apply order: §1 migrations → §2 models/trait → §3 mail/job → §4 service → §5 guards → §6 controller → §7 permissions/env → §8 routes. Then `config:clear`, `migrate --force`, seed, `queue:restart` (full commands in runbook).

---

## 1. NEW — `database/migrations/2026_09_22_000001_add_archive_columns_to_meters_table.php`

Purpose: unified archive columns on `meters`. All existing rows default `is_archived=false`, so nothing freezes on deploy.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * Adds unified archive support to meters (both meter_type
     * promise_tracker and promise_ring). Manual archive only;
     * tenure_date_end passing does NOT auto-freeze.
     */
    public function up(): void
    {
        Schema::table('meters', function (Blueprint $table) {
            $table->boolean('is_archived')->default(false)->after('meter_type');
            $table->timestamp('archived_at')->nullable()->after('is_archived');
            $table->unsignedBigInteger('archived_by')->nullable()->after('archived_at');
            $table->uuid('parent_meter_id')->nullable()->after('archived_by');

            $table->index('is_archived');
            $table->index('parent_meter_id');
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::table('meters', function (Blueprint $table) {
            $table->dropIndex(['is_archived']);
            $table->dropIndex(['parent_meter_id']);
            $table->dropColumn(['is_archived', 'archived_at', 'archived_by', 'parent_meter_id']);
        });
    }
};
```

## 2. NEW — `database/migrations/2026_09_22_000002_create_meter_archive_requests_table.php`

Purpose: approval queue. Meter stays editable while `pending`.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * Approval queue for meter archive / unarchive.
     * Only the admin whose email matches ARCHIVE_APPROVAL_EMAIL
     * can approve or reject. Meter stays editable while pending.
     */
    public function up(): void
    {
        Schema::create('meter_archive_requests', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->uuid('meter_id');
            $table->enum('type', ['archive', 'unarchive'])->default('archive');
            $table->unsignedBigInteger('requested_by');
            $table->string('approver_email');
            $table->enum('status', ['pending', 'approved', 'rejected'])->default('pending');
            $table->text('reason')->nullable();
            $table->unsignedBigInteger('decided_by')->nullable();
            $table->timestamp('decided_at')->nullable();
            $table->timestamps();

            $table->index('meter_id');
            $table->index('status');
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('meter_archive_requests');
    }
};
```

## 3. EDIT — `app/Models/Meter.php` (full file after edit, 74 lines)

What: casts + `parent/children/archiveRequests` + `isArchived/isEditable` + `scopeActive/scopeArchived`.

```php
<?php

namespace App\Models;

use App\Traits\UUID;
use Carbon\Carbon;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;


class Meter extends Model
{
    use HasFactory ,UUID ,SoftDeletes;

    protected $guarded = ['id'];

    protected $dates = ['tenure_date_start', 'tenure_date_end'];

    protected $casts = [
        'is_archived' => 'boolean',
        'archived_at' => 'datetime',
    ];

    public function isCurrent(): bool
    {
        $now = Carbon::now();

        $startDate = Carbon::parse($this->tenure_date_start);
        $endDate = Carbon::parse($this->tenure_date_end);

        return $now->between($startDate, $endDate);
    }

    public function promises(): \Illuminate\Database\Eloquent\Relations\HasMany
    {
        return $this->hasMany(Promise::class);
    }

    public function parent(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Meter::class, 'parent_meter_id');
    }

    public function children(): \Illuminate\Database\Eloquent\Relations\HasMany
    {
        return $this->hasMany(Meter::class, 'parent_meter_id');
    }

    public function archiveRequests(): \Illuminate\Database\Eloquent\Relations\HasMany
    {
        return $this->hasMany(MeterArchiveRequest::class, 'meter_id');
    }

    public function isArchived(): bool
    {
        return (bool) $this->is_archived;
    }

    public function isEditable(): bool
    {
        return ! $this->is_archived;
    }

    public function scopeActive($query)
    {
        return $query->where('is_archived', false);
    }

    public function scopeArchived($query)
    {
        return $query->where('is_archived', true);
    }
}
```

## 4. NEW — `app/Models/MeterArchiveRequest.php`

```php
<?php

namespace App\Models;

use App\Traits\UUID;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class MeterArchiveRequest extends Model
{
    use HasFactory, UUID;

    protected $guarded = ['id'];

    protected $casts = [
        'decided_at' => 'datetime',
    ];

    const TYPE_ARCHIVE = 'archive';
    const TYPE_UNARCHIVE = 'unarchive';

    const STATUS_PENDING = 'pending';
    const STATUS_APPROVED = 'approved';
    const STATUS_REJECTED = 'rejected';

    public function meter(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Meter::class, 'meter_id');
    }

    public function requester(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Admin::class, 'requested_by');
    }

    public function decider(): \Illuminate\Database\Eloquent\Relations\BelongsTo
    {
        return $this->belongsTo(Admin::class, 'decided_by');
    }

    public function isPending(): bool
    {
        return $this->status === self::STATUS_PENDING;
    }
}
```

## 5. NEW — `app/Traits/GuardsMeterArchive.php`

Purpose: single 403 helper reused by all write paths. Returns `null` when editable or meter not found (caller handles 404; reads never blocked).

```php
<?php

namespace App\Traits;

use App\Models\Meter;

trait GuardsMeterArchive
{
    /**
     * Return a standard 403 service error if the meter is archived.
     * Returns null when the meter is editable (or not found — let
     * the caller handle not-found so reads still work).
     */
    protected function archivedMeterError($meter): ?array
    {
        if ($meter instanceof Meter && (bool) $meter->is_archived) {
            return [
                'status' => false,
                'message' => 'Meter is archived and read-only',
                'errors' => ['Meter is archived and read-only'],
                'statusCode' => 403,
                'data' => null,
            ];
        }

        return null;
    }

    protected function findMeter(string $meterId): ?Meter
    {
        return Meter::where('id', $meterId)->first();
    }
}
```

## 6. NEW — `app/Mail/MeterArchiveMail.php`

```php
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class MeterArchiveMail extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    public $payload;

    /**
     * @param array $payload ['title','body','action_url','action_label']
     */
    public function __construct(array $payload)
    {
        $this->payload = $payload;
    }

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: $this->payload['title'] ?? 'Meter archive request',
        );
    }

    public function content(): Content
    {
        return new Content(
            markdown: 'emails.meter-archive',
            with: ['payload' => $this->payload],
        );
    }

    public function attachments(): array
    {
        return [];
    }
}
```

## 7. NEW — `resources/views/emails/meter-archive.blade.php`

```blade
<x-mail::message>
<p>Hello,</p>

<p>{{ $payload['body'] ?? '' }}</p>

@if(!empty($payload['details']))
<x-mail::panel>
{{ $payload['details'] }}
</x-mail::panel>
@endif

@if(!empty($payload['action_url']))
<x-mail::button :url="$payload['action_url']">
{{ $payload['action_label'] ?? 'Review request' }}
</x-mail::button>
@endif

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

## 8. NEW — `app/Jobs/SendMeterArchiveEmail.php`

Purpose: queued mail (dispatched `->onQueue('mail-high')`, same worker as promise mails).

```php
<?php

namespace App\Jobs;

use App\Mail\MeterArchiveMail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class SendMeterArchiveEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $emails;
    public $payload;

    public function __construct(array $emails, array $payload)
    {
        $this->emails = $emails;
        $this->payload = $payload;
    }

    public function handle(): void
    {
        foreach ($this->emails as $email) {
            Mail::to($email)->send(new MeterArchiveMail($this->payload));
        }
    }
}
```

## 9. NEW — `app/Services/Admin/Meter/MeterArchiveService.php` (full file)

Purpose: request/approve/reject/resend/pending. Key rules: duplicate-pending → 409; missing `ARCHIVE_APPROVAL_EMAIL` → 500; approve/reject requires `strtolower(trim(admin.email)) === strtolower(trim(request.approver_email))` else 403; approve flips `is_archived/archived_at/archived_by`; meter editable until approval; mails on request + decision.

```php
<?php

namespace App\Services\Admin\Meter;

use App\Jobs\SendMeterArchiveEmail;
use App\Models\Admin;
use App\Models\Meter;
use App\Models\MeterArchiveRequest;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class MeterArchiveService
{
    public function resolveApprovalEmail(): ?string
    {
        $email = env('ARCHIVE_APPROVAL_EMAIL');

        if (! $email) {
            return null;
        }

        $email = trim($email);

        return $email !== '' ? $email : null;
    }

    public function requestArchive(string $meterId, string $type, int $requestedBy, ?string $reason = null): array
    {
        try {
            $type = strtolower($type) === MeterArchiveRequest::TYPE_UNARCHIVE
                ? MeterArchiveRequest::TYPE_UNARCHIVE
                : MeterArchiveRequest::TYPE_ARCHIVE;

            $meter = Meter::where('id', $meterId)->first();

            if (! $meter) {
                return $this->error('Meter not found', 404);
            }

            if ($type === MeterArchiveRequest::TYPE_ARCHIVE && (bool) $meter->is_archived) {
                return $this->error('Meter is already archived', 409);
            }

            if ($type === MeterArchiveRequest::TYPE_UNARCHIVE && ! (bool) $meter->is_archived) {
                return $this->error('Meter is not archived', 409);
            }

            $existing = MeterArchiveRequest::where('meter_id', $meterId)
                ->where('type', $type)
                ->where('status', MeterArchiveRequest::STATUS_PENDING)
                ->first();

            if ($existing) {
                return [
                    'status' => false,
                    'message' => 'A pending ' . $type . ' request already exists for this meter',
                    'errors' => ['A pending ' . $type . ' request already exists for this meter'],
                    'statusCode' => 409,
                    'data' => $existing,
                ];
            }

            $approvalEmail = $this->resolveApprovalEmail();

            if (! $approvalEmail) {
                return $this->error('ARCHIVE_APPROVAL_EMAIL is not configured', 500);
            }

            $requester = Admin::where('id', $requestedBy)->first();

            DB::beginTransaction();

            $archiveRequest = new MeterArchiveRequest();
            $archiveRequest->meter_id = $meter->id;
            $archiveRequest->type = $type;
            $archiveRequest->requested_by = $requestedBy;
            $archiveRequest->approver_email = $approvalEmail;
            $archiveRequest->status = MeterArchiveRequest::STATUS_PENDING;
            $archiveRequest->reason = $reason;
            $archiveRequest->save();

            DB::commit();

            $this->sendRequestMail($archiveRequest, $meter, $requester);

            return [
                'status' => true,
                'message' => ucfirst($type) . ' request submitted. Awaiting approval.',
                'data' => $archiveRequest->fresh()->load('meter'),
                'statusCode' => 200,
                'errors' => [],
            ];
        } catch (\Exception $th) {
            DB::rollBack();
            Log::error('Meter archive request failed: ' . $th->getMessage());

            return $this->error('An issue occurred', 500, [$th->getMessage()]);
        }
    }

    public function approve(string $requestId, Admin $approver): array
    {
        try {
            $archiveRequest = MeterArchiveRequest::where('id', $requestId)->with('meter')->first();

            if (! $archiveRequest) {
                return $this->error('Archive request not found', 404);
            }

            if (! $archiveRequest->isPending()) {
                return $this->error('Request has already been decided', 409);
            }

            $check = $this->assertApprover($archiveRequest, $approver);
            if ($check) {
                return $check;
            }

            $meter = $archiveRequest->meter;

            if (! $meter) {
                return $this->error('Meter not found', 404);
            }

            DB::beginTransaction();

            if ($archiveRequest->type === MeterArchiveRequest::TYPE_ARCHIVE) {
                $meter->update([
                    'is_archived' => true,
                    'archived_at' => now(),
                    'archived_by' => $approver->id,
                ]);
            } else {
                $meter->update([
                    'is_archived' => false,
                    'archived_at' => null,
                    'archived_by' => null,
                ]);
            }

            $archiveRequest->update([
                'status' => MeterArchiveRequest::STATUS_APPROVED,
                'decided_by' => $approver->id,
                'decided_at' => now(),
            ]);

            DB::commit();

            $this->sendDecisionMail($archiveRequest->fresh()->load('meter'), true, null);

            return [
                'status' => true,
                'message' => 'Meter ' . $archiveRequest->type . ' approved',
                'data' => $archiveRequest->fresh()->load('meter'),
                'statusCode' => 200,
                'errors' => [],
            ];
        } catch (\Exception $th) {
            DB::rollBack();
            Log::error('Meter archive approve failed: ' . $th->getMessage());

            return $this->error('An issue occurred', 500, [$th->getMessage()]);
        }
    }

    public function reject(string $requestId, Admin $approver, ?string $reason = null): array
    {
        try {
            $archiveRequest = MeterArchiveRequest::where('id', $requestId)->with('meter')->first();

            if (! $archiveRequest) {
                return $this->error('Archive request not found', 404);
            }

            if (! $archiveRequest->isPending()) {
                return $this->error('Request has already been decided', 409);
            }

            $check = $this->assertApprover($archiveRequest, $approver);
            if ($check) {
                return $check;
            }

            DB::beginTransaction();

            $archiveRequest->update([
                'status' => MeterArchiveRequest::STATUS_REJECTED,
                'decided_by' => $approver->id,
                'decided_at' => now(),
                'reason' => $reason ?? $archiveRequest->reason,
            ]);

            DB::commit();

            $this->sendDecisionMail($archiveRequest->fresh()->load('meter'), false, $reason);

            return [
                'status' => true,
                'message' => 'Meter ' . $archiveRequest->type . ' rejected',
                'data' => $archiveRequest->fresh()->load('meter'),
                'statusCode' => 200,
                'errors' => [],
            ];
        } catch (\Exception $th) {
            DB::rollBack();
            Log::error('Meter archive reject failed: ' . $th->getMessage());

            return $this->error('An issue occurred', 500, [$th->getMessage()]);
        }
    }

    public function resend(string $requestId): array
    {
        try {
            $archiveRequest = MeterArchiveRequest::where('id', $requestId)->with('meter')->first();

            if (! $archiveRequest) {
                return $this->error('Archive request not found', 404);
            }

            if (! $archiveRequest->isPending()) {
                return $this->error('Only pending requests can be re-sent', 409);
            }

            $requester = Admin::where('id', $archiveRequest->requested_by)->first();

            $this->sendRequestMail($archiveRequest, $archiveRequest->meter, $requester, true);

            return [
                'status' => true,
                'message' => 'Approval reminder re-sent',
                'data' => $archiveRequest,
                'statusCode' => 200,
                'errors' => [],
            ];
        } catch (\Exception $th) {
            Log::error('Meter archive resend failed: ' . $th->getMessage());

            return $this->error('An issue occurred', 500, [$th->getMessage()]);
        }
    }

    public function pending(): array
    {
        try {
            $requests = MeterArchiveRequest::where('status', MeterArchiveRequest::STATUS_PENDING)
                ->with(['meter', 'requester'])
                ->orderBy('created_at', 'desc')
                ->get();

            return [
                'status' => true,
                'message' => 'Pending archive requests fetched',
                'data' => $requests,
                'statusCode' => 200,
                'errors' => [],
            ];
        } catch (\Exception $th) {
            return $this->error('An issue occurred', 500, [$th->getMessage()]);
        }
    }

    /**
     * Only the admin whose email matches the request approver_email
     * (which equals ARCHIVE_APPROVAL_EMAIL at request time) can decide.
     */
    protected function assertApprover(MeterArchiveRequest $archiveRequest, Admin $approver): ?array
    {
        $approverEmail = strtolower(trim((string) $approver->email));
        $requiredEmail = strtolower(trim((string) $archiveRequest->approver_email));

        if ($approverEmail === '' || $approverEmail !== $requiredEmail) {
            return $this->error('Only ' . $archiveRequest->approver_email . ' can approve this request', 403);
        }

        return null;
    }

    protected function sendRequestMail(MeterArchiveRequest $archiveRequest, ?Meter $meter, ?Admin $requester, bool $isReminder = false): void
    {
        $approvalEmail = $archiveRequest->approver_email;
        $meterLabel = $meter ? $meter->name . ' (' . $meter->meter_name . ')' : $archiveRequest->meter_id;
        $requesterLabel = $requester ? $requester->name . ' (' . $requester->email . ')' : 'Admin #' . $archiveRequest->requested_by;

        $title = ($isReminder ? 'Reminder: ' : '') . 'Meter ' . $archiveRequest->type . ' approval needed';

        SendMeterArchiveEmail::dispatch([$approvalEmail], [
            'title' => $title,
            'body' => $requesterLabel . ' requested to ' . $archiveRequest->type . ' meter ' . $meterLabel . '. The meter stays editable until you approve. Only you can approve this request.',
            'details' => 'Request ID: ' . $archiveRequest->id . ' | Meter ID: ' . $archiveRequest->meter_id . ' | Type: ' . $archiveRequest->type,
            'action_url' => $this->reviewUrl($archiveRequest->id),
            'action_label' => 'Review request',
        ])->onQueue('mail-high');
    }

    protected function sendDecisionMail(MeterArchiveRequest $archiveRequest, bool $approved, ?string $reason): void
    {
        $requester = Admin::where('id', $archiveRequest->requested_by)->first();

        if (! $requester) {
            return;
        }

        $meterLabel = $archiveRequest->meter
            ? $archiveRequest->meter->name . ' (' . $archiveRequest->meter->meter_name . ')'
            : $archiveRequest->meter_id;

        $title = 'Meter ' . $archiveRequest->type . ' ' . ($approved ? 'approved' : 'rejected');

        $body = 'Your ' . $archiveRequest->type . ' request for meter ' . $meterLabel
            . ' was ' . ($approved ? 'approved. The meter is now ' . ($archiveRequest->type === 'archive' ? 'archived and read-only.' : 'active and editable.') : 'rejected. The meter remains editable.');

        if ($reason) {
            $body .= ' Reason: ' . $reason;
        }

        SendMeterArchiveEmail::dispatch([$requester->email], [
            'title' => $title,
            'body' => $body,
            'details' => 'Request ID: ' . $archiveRequest->id,
            'action_url' => null,
            'action_label' => null,
        ])->onQueue('mail-high');
    }

    protected function reviewUrl(string $requestId): ?string
    {
        $base = rtrim((string) env('APP_URL', ''), '/');

        if ($base === '') {
            return null;
        }

        return $base . '/admin/meter-archive/' . $requestId;
    }

    protected function error(string $message, int $code, array $errors = []): array
    {
        return [
            'status' => false,
            'message' => $message,
            'errors' => $errors !== [] ? $errors : [$message],
            'statusCode' => $code,
            'data' => null,
        ];
    }
}
```

## 10. EDIT — `app/Services/Admin/Meter/MeterService.php` (2 hunks)

Hunk A — `createMeter()`: new meters always active + accept optional `parent_meter_id` (tenure-2 links tenure-1). Find:

```php
            $meter->meter_type         = $data['type_of_meter'];
            $meter->save();
```

Replace with:

```php
            $meter->meter_type         = $data['type_of_meter'];
            $meter->is_archived        = false;
            $meter->parent_meter_id    = isset($data['parent_meter_id']) && $data['parent_meter_id'] !== 'null'
                ? $data['parent_meter_id']
                : null;
            $meter->save();
```

Hunk B — `getCurrentMeter()`: archived never auto-picked; explicit ID still readable. Find:

```php
                    ->where('meter_type',$type)
                    ->where('tenure_date_start', '<=', now())
```

Replace with:

```php
                    ->where('meter_type',$type)
                    ->where('is_archived', false)
                    ->where('tenure_date_start', '<=', now())
```

## 11. EDIT — `app/Services/Admin/Promise/PromiseService.php` (guards)

Step 1 — add trait. Find:

```php
use App\Models\Update;
use App\Traits\ImageTrait;
```

Replace with:

```php
use App\Models\Update;
use App\Traits\GuardsMeterArchive;
use App\Traits\ImageTrait;
```

Find:

```php
class PromiseService
{
    use ImageTrait;
```

Replace with:

```php
class PromiseService
{
    use ImageTrait, GuardsMeterArchive;
```

Step 2 — `createPromise()`: block on archived target meter. After validation `}`, before `DB::beginTransaction();`, insert:

```php
            $targetMeter = $this->findMeter($data['meter_id'] ?? '');
            if ($blocked = $this->archivedMeterError($targetMeter)) {
                return $blocked;
            }
```

Step 3 — `updatePromise($data,$id)`: after the `Promise not found` 404 block, insert:

```php
            if ($blocked = $this->archivedMeterError($promise->meter ?? $this->findMeter($promise->meter_id))) {
                return $blocked;
            }
```

Step 4 — `deletePromise($id)`: after 404 block, before `$promise->delete();`, insert:

```php
            if ($blocked = $this->archivedMeterError($this->findMeter($promise->meter_id))) {
                return $blocked;
            }
```

Step 5 — `createPromiseUpdates()`: after `Promise Not Found` 404 block, insert:

```php
           if ($blocked = $this->archivedMeterError($this->findMeter($promise->meter_id))) {
               DB::rollBack();
               return $blocked;
           }
```

Step 6 — `editPromiseUpdates()`: after `$update = Update::where('id',$update_id)->first();`, insert null-check + meter guard, then keep `$update->update(...)`:

```php
            $update =  Update::where('id',$update_id)->first();

            if(!$update){
                DB::rollBack();
                return [
                    'status' => false,
                    'message' => 'Update not found',
                    'errors' => ['Update not found'],
                    'statusCode' => 404,
                    'data'=> null
                ];
            }

            $updatePromise = Promise::where('id',$update->promise_id)->first();

            if ($updatePromise && ($blocked = $this->archivedMeterError($this->findMeter($updatePromise->meter_id)))) {
                DB::rollBack();
                return $blocked;
            }
```

Step 7 — `publishPromise()`: for `promise/ring` add 404 + archived check before `$data->update(['published_status' => 'published'])`; for `promise_update` add 404 + parent-meter check before `$data->update(['is_published' => 1])`. (Null-checks added because original code assumed rows exist.)

Step 8 — `unPublishPromise()`: same pattern for both branches — 404 + archived check before any `update()`.

Step 9 — `statusPromiseUpdates()`: after `$promise = Promise::where('id', $promise_id)...`, insert 404 + archived check.

Step 10 — `deletePromiseUpdate()`: after `$promiseUpdate = Update::...`, insert `if(!$promiseUpdate)` 404; after `$findPromise = ...`, insert archived check with `DB::rollBack()`.

All guards return the shared envelope: `403 {status:false, message:"Meter is archived and read-only"}`.

## 12. EDIT — `app/Services/Api/Promise/PromiseService.php` (public reads)

- `getMeterByType()` presidential: added `->where('is_archived', false)` to the tenure-bracket pick.
- `getMeterByType()` state (both branches) + legislative: added `->where('is_archived', false)` + `orderBy('created_at','desc')` for the single-state branch so "current" means latest active.
- `promiseRing($state)`: `$meterIds` query added `->where('is_archived', false)` — Ring list shows active tenure only; archived stays reachable by direct meter ID.
- `getPromiseRingStates()` + `getPromiseDataByMeterId()` selects extended with `is_archived,parent_meter_id` so frontend can badge/banner without extra calls. Example:

```php
$meter = Meter::where('promise_tracker_id',$promiseTrackerId)
    ->select('id','name','meter_name','tenure_date_start','tenure_date_end','meter_img','chamber','state_id','is_archived','parent_meter_id')
    ->get();
```

## 13. EDIT — `app/Imports/PromisesImport.php`

At the top of `model(array $row)`, insert (archived meters skip silently + warn):

```php
        $meter = Meter::where('id', $this->meterId)->first();

        if ($meter && (bool) $meter->is_archived) {
            Log::warning('Skipping import row for archived meter: ' . $this->meterId);

            return null;
        }
```

## 14. EDIT — `app/Http/Controllers/Admin/Promise/Promise.php`

`importPromises()`: pre-check before touching Excel. After `$request->validate(...)`, insert:

```php
        $meter = \App\Models\Meter::where('id', $meterId)->first();

        if ($meter && (bool) $meter->is_archived) {
            return $this->sendError('Meter is archived and read-only', ['Meter is archived and read-only'], 403);
        }
```

## 15. NEW — `app/Http/Controllers/Admin/Meter/MeterArchive.php` (full file)

```php
<?php

namespace App\Http\Controllers\Admin\Meter;

use App\Http\Controllers\Controller;
use App\Services\Admin\Meter\MeterArchiveService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class MeterArchive extends Controller
{
    protected MeterArchiveService $archiveService;

    public function __construct(MeterArchiveService $archiveService)
    {
        $this->archiveService = $archiveService;
    }

    public function requestArchive(Request $request, $meterId): JsonResponse
    {
        $type = $request->input('type', 'archive');
        $reason = $request->input('reason');

        $admin = auth()->guard('admin')->user();

        $result = $this->archiveService->requestArchive($meterId, $type, $admin->id, $reason);

        return $result['status']
            ? $this->sendResponse($result['data'], $result['message'])
            : $this->sendError($result['message'], $result['errors'], $result['statusCode']);
    }

    public function approve($requestId): JsonResponse
    {
        $admin = auth()->guard('admin')->user();

        $result = $this->archiveService->approve($requestId, $admin);

        return $result['status']
            ? $this->sendResponse($result['data'], $result['message'])
            : $this->sendError($result['message'], $result['errors'], $result['statusCode']);
    }

    public function reject(Request $request, $requestId): JsonResponse
    {
        $admin = auth()->guard('admin')->user();

        $result = $this->archiveService->reject($requestId, $admin, $request->input('reason'));

        return $result['status']
            ? $this->sendResponse($result['data'], $result['message'])
            : $this->sendError($result['message'], $result['errors'], $result['statusCode']);
    }

    public function resend($requestId): JsonResponse
    {
        $result = $this->archiveService->resend($requestId);

        return $result['status']
            ? $this->sendResponse($result['data'], $result['message'])
            : $this->sendError($result['message'], $result['errors'], $result['statusCode']);
    }

    public function pending(): JsonResponse
    {
        $result = $this->archiveService->pending();

        return $result['status']
            ? $this->sendResponse($result['data'], $result['message'])
            : $this->sendError($result['message'], $result['errors'], $result['statusCode']);
    }
}
```

## 16. EDIT — `database/seeders/PermissionSeeder.php`

Under `//Meter`, add (fresh installs only; live uses §17):

```php
        Permission::create(['guard_name' => 'admin','name' => 'archive-meter']);
        Permission::create(['guard_name' => 'admin','name' => 'unarchive-meter']);
```

## 17. NEW — `database/seeders/MeterArchivePermissionSeeder.php` (live-safe, re-runnable)

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;

/**
 * Live-safe seeder for the meter archive feature.
 *
 * Run on live AFTER migrating:
 *   php artisan db:seed --class=MeterArchivePermissionSeeder
 *
 * Creates archive-meter / unarchive-meter (if missing) and
 * grants both to the Super Administrator role (if present).
 * Safe to re-run: uses firstOrCreate + syncWithoutDetaching.
 */
class MeterArchivePermissionSeeder extends Seeder
{
    public function run(): void
    {
        $names = ['archive-meter', 'unarchive-meter'];

        foreach ($names as $name) {
            Permission::firstOrCreate(
                ['name' => $name, 'guard_name' => 'admin'],
                ['name' => $name, 'guard_name' => 'admin']
            );
        }

        $role = Role::where('name', 'Super Administrator')->where('guard_name', 'admin')->first();

        if ($role) {
            $role->syncPermissions(array_merge(
                $role->permissions->pluck('name')->toArray(),
                $names
            ));
        }
    }
}
```

## 18. EDIT — `.env.example` (append)

```
PROMISE_NOTIFICATION_EMAIL=""
# Only this admin email can approve meter archive / unarchive requests.
# Must match an existing admins.email (Super Administrator).
ARCHIVE_APPROVAL_EMAIL=""
```

Live: add `ARCHIVE_APPROVAL_EMAIL=habib@advokc.ng` (or the real owner inbox) to `/var/www/promise_tracker/.env`, then `php artisan config:clear`.

## 19. NEW — `routes_snippet/meter-archive-routes.php` (paste into LIVE `routes/api.php`)

Local copy has no `routes/` dir (architecture gap #1); live does. Paste inside the admin meter group next to `meter/create`:

```php
use App\Http\Controllers\Admin\Meter\MeterArchive;
use Illuminate\Support\Facades\Route;

// --- PASTE START ---
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
// --- PASTE END ---
```

Then: `php artisan route:clear && php artisan config:clear && php artisan route:list | grep -i archive` (expect 5 rows).

---

## 20. Verify after applying

```bash
php -l database/migrations/2026_09_22_000001_add_archive_columns_to_meters_table.php
php -l database/migrations/2026_09_22_000002_create_meter_archive_requests_table.php
php -l app/Models/Meter.php
php -l app/Models/MeterArchiveRequest.php
php -l app/Traits/GuardsMeterArchive.php
php -l app/Mail/MeterArchiveMail.php
php -l app/Jobs/SendMeterArchiveEmail.php
php -l app/Services/Admin/Meter/MeterArchiveService.php
php -l app/Services/Admin/Meter/MeterService.php
php -l "app/Services/Admin/Promise/PromiseService.php"
php -l "app/Services/Api/Promise/PromiseService.php"
php -l app/Http/Controllers/Admin/Meter/MeterArchive.php
php -l "app/Http/Controllers/Admin/Promise/Promise.php"
php -l app/Imports/PromisesImport.php
php -l database/seeders/PermissionSeeder.php
php -l database/seeders/MeterArchivePermissionSeeder.php
```

Live deploy + curl checks + rollback: see `meter-archive-runbook.md` §§4–9.
