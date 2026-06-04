## Scope
This file owns only:
- Canonical current-state documentation for the binders domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Binders

The binders domain manages physical binders that hold client tax documents. Each binder belongs to one `client_record`, receives material intakes over time, transitions through a defined lifecycle, and is eventually handed over to the client. All materials from all of a client's businesses are stored in the same binder; the material type is tracked at the `BinderIntakeMaterial` level, not at binder level.

Last verified against code + backend/openapi.json: 2026-06-04.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/binders | List active binders with filters, sorting, pagination |
| GET | /api/v1/binders/open | List binders not yet handed over |
| GET | /api/v1/binders/{binder_id} | Get single binder by ID |
| DELETE | /api/v1/binders/{binder_id} | Soft-delete binder (ADVISOR only) |
| POST | /api/v1/binders/receive | Receive material (find or create binder + intake) |
| POST | /api/v1/binders/{binder_id}/receive-material | Log a material receipt on existing binder |
| POST | /api/v1/binders/{binder_id}/mark-full | Mark binder capacity full |
| POST | /api/v1/binders/{binder_id}/reopen-capacity | Reopen binder capacity |
| POST | /api/v1/binders/{binder_id}/mark-ready-for-handover | Mark single binder ready for handover |
| POST | /api/v1/binders/mark-ready-for-handover-bulk | Mark all eligible binders for a client ready (by period cutoff) |
| POST | /api/v1/binders/{binder_id}/revert-ready-for-handover | Revert ready-for-handover back to in-office |
| POST | /api/v1/binders/{binder_id}/handover-to-client | Hand over single binder to client |
| POST | /api/v1/binders/handover-to-client-bulk | Hand over multiple binders in one grouped event |
| GET | /api/v1/binders/{binder_id}/history | Lifecycle audit log for binder |
| GET | /api/v1/binders/{binder_id}/intakes | All material intakes for binder |
| PATCH | /api/v1/binders/{binder_id}/intakes/{intake_id} | Edit an existing intake (fields + cross-client transfer) |
| GET | /api/v1/clients/{client_record_id}/binders | All binders for a specific client |

All paths confirmed in `backend/openapi.json`. All endpoints require role `ADVISOR` or `SECRETARY`; `DELETE /binders/{binder_id}` requires `ADVISOR`.

## Model & fields

### Binder (`binders` table)
Cite: `backend/app/binders/models/binder.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | autoincrement |
| client_record_id | int FK→client_records.id | no | indexed |
| binder_number | str | no | format: `{office_client_number}/{sequence}`, unique per active client_record (partial index where deleted_at IS NULL) |
| period_start | date | yes | derived from first material's `period_year`/`period_month_start`; NULL until first intake |
| period_end | date | yes | NULL = active/open binder; set to last material period when binder is closed or to `handed_over_at` on handover |
| location_status | enum | no | `in_office` \| `ready_for_handover` \| `handed_over`; default `in_office` |
| capacity_status | enum | no | `open` \| `full`; default `open` |
| ready_for_handover_at | datetime | yes | timestamp set when transitioning to `ready_for_handover` |
| handed_over_at | date | yes | date of physical handover |
| handover_recipient_name | str | yes | name of person who received binders on client side |
| notes | text | yes | physical logistics info |
| created_at | datetime | no | UTC |
| created_by | int FK→users.id | no | |
| deleted_at | datetime | yes | soft delete |
| deleted_by | int FK→users.id | yes | soft delete actor |

### BinderIntake (`binder_intakes` table)
Cite: `backend/app/binders/models/binder_intake.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| binder_id | int FK→binders.id | no | CASCADE delete; indexed |
| received_at | date | no | date material was received |
| received_by | int FK→users.id | no | office staff who received |
| notes | text | yes | free-text event notes |
| created_at | datetime | no | UTC |

### BinderIntakeMaterial (`binder_intake_materials` table)
Cite: `backend/app/binders/models/binder_intake_material.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| intake_id | int FK→binder_intakes.id | no | indexed |
| business_id | int FK→businesses.id | yes | nullable = client-level generic material |
| material_type | enum | no | see MaterialType enum below |
| annual_report_id | int FK→annual_reports.id | yes | only for `annual_report` type |
| vat_report_id | int FK→vat_work_items.id | yes | only for `vat` type |
| period_year | int | no | reporting period year |
| period_month_start | int | no | 1–12 |
| period_month_end | int | no | 1–12; equals `period_month_start` for monthly |
| description | text | yes | optional free-text note |
| created_at | datetime | no | UTC |

### BinderHandover (`binder_handovers` table)
Cite: `backend/app/binders/models/binder_handover.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| client_record_id | int FK→client_records.id | no | indexed |
| received_by_name | str | no | name of person who physically received |
| handed_over_at | date | no | date of physical handover |
| until_period_year | int | no | period cutoff: all binders up to this year covered |
| until_period_month | int | no | 1–12 |
| notes | text | yes | |
| created_by | int FK→users.id | no | |
| created_at | datetime | no | UTC |

### BinderHandoverBinder (`binder_handover_binders` table)
Association: handover event ↔ specific binders. Unique index on `(handover_id, binder_id)`.

### BinderLifecycleLog (`binder_lifecycle_logs` table)
Cite: `backend/app/binders/models/binder_lifecycle_log.py`

One row per changed field per lifecycle transition.

| Column | Type | Nullable |
|--------|------|----------|
| id | int PK | no |
| binder_id | int FK→binders.id | no |
| field_name | str | no | `location_status` or `capacity_status` |
| old_value | str | no | |
| new_value | str | no | |
| changed_by_user_id | int FK→users.id | no | |
| changed_at | datetime | no | UTC |
| notes | text | yes | reason/context |

### BinderIntakeEditLog (`binder_intake_edit_logs` table)
Cite: `backend/app/binders/models/binder_intake_edit_log.py`

Field-level audit trail for edits to `BinderIntake` and its `BinderIntakeMaterial` rows. One row per changed field per edit.

## Enums / statuses

### BinderLocationStatus
Cite: `backend/app/binders/models/binder.py:27-30`

| Value | Meaning |
|-------|---------|
| `in_office` | Binder is physically in the office |
| `ready_for_handover` | Binder is prepared for handover to client |
| `handed_over` | Binder was physically handed to client |

### BinderCapacityStatus
Cite: `backend/app/binders/models/binder.py:33-35`

| Value | Meaning |
|-------|---------|
| `open` | Binder can still receive material (capacity only; not alone sufficient for intake) |
| `full` | Binder is full |

### MaterialType
Cite: `backend/app/binders/models/binder_intake_material.py:18-28`

`vat`, `income_tax`, `annual_report`, `salary`, `bookkeeping`, `national_insurance`, `capital_declaration`, `pension_and_insurance`, `corporate_docs`, `tax_assessment`, `other`

## Domain rules & invariants

### Lifecycle state machine
Cite: `backend/app/binders/services/binder_lifecycle_service.py`

All `location_status` and `capacity_status` mutations go through `BinderLifecycleService` exclusively. No router, repository, or cross-domain service may mutate these fields directly.

**Intake eligibility:** Only `location_status=in_office` AND `capacity_status=open` is eligible for material intake. `capacity_status=open` alone is not sufficient.

**Available actions per state** (derived by `BinderLifecycleService.get_available_action_keys_for_state`):

| location_status | capacity_status | Available actions |
|----------------|----------------|-------------------|
| `in_office` | `open` | `receive_material`, `mark_full`, `mark_ready_for_handover` |
| `in_office` | `full` | `reopen_capacity`, `mark_ready_for_handover` |
| `ready_for_handover` | any | `revert_ready_for_handover`, `handover_to_client` |
| `handed_over` | any | (none) |

**Transition rules:**

- **receive_material**: allowed for `in_office` only. The `allow_full_in_office` flag permits receipt on a full in-office binder when called via `BinderIntakeService` (old-period material routing). Error: `BINDER.NOT_INTAKE_ELIGIBLE`.
- **mark_full**: `in_office + open` → `in_office + full`. Errors: `BINDER.ALREADY_FULL`, `BINDER.CAPACITY_CHANGE_NOT_ALLOWED`.
- **reopen_capacity**: `in_office + full` → `in_office + open`. Errors: `BINDER.NOT_FULL`, `BINDER.CAPACITY_CHANGE_NOT_ALLOWED`.
- **mark_ready_for_handover**: `in_office + any` → `ready_for_handover + (preserved capacity)`. Triggers an auto-notification (`NotificationTrigger.BINDER_READY_FOR_HANDOVER`) with idempotency key. Error: `BINDER.INVALID_LOCATION_TRANSITION`.
- **revert_ready_for_handover**: `ready_for_handover` → `in_office + (preserved capacity)`. Error: `BINDER.INVALID_LOCATION_TRANSITION`.
- **handover_to_client**: `ready_for_handover` → `handed_over`. Sets `handed_over_at` (defaults to today), `handover_recipient_name`, and backfills `period_end` if NULL. Errors: `BINDER.NOT_READY_FOR_HANDOVER`, `BINDER.ALREADY_HANDED_OVER`.

**Post-handover rule:** A handed-over binder is never reused for new material. If new material arrives after handover, a new binder is created.

### Intake flow (`BinderIntakeService.receive`)
Cite: `backend/app/binders/services/binder_intake_service.py`

1. Client record must be active (`assert_client_record_is_active`).
2. If client has businesses and all are non-active → `BINDER.CLIENT_LOCKED`.
3. Finds the current intake-eligible binder (`in_office + open`). If incoming material period is older than the active binder's `period_start`, tries to route to a matching older in-office binder before falling back to the active one.
4. If `open_new_binder=True` and an active binder exists: closes it (sets `period_end`, marks full) and creates a new binder.
5. New binder requires `office_client_number` on the client record; `binder_number` = `"{office_client_number}/{sequence}"`. Error: `BINDER.OFFICE_NUMBER_MISSING`.
6. Old-period note guard: if any material period is older than binder's `period_start`, the intake `notes` field must be non-empty. Error: `BINDER.OLD_PERIOD_NOTE_REQUIRED`.
7. After creating intake materials, backfills `binder.period_start` from the first material if still NULL.
8. VAT side-effect: linked VAT work items in `PENDING_MATERIALS` status are auto-advanced to `MATERIAL_RECEIVED`.

### Grouped handover (`BinderHandoverService.create_handover`)
Cite: `backend/app/binders/services/binder_handover_service.py`

All binders in the list must belong to `client_record_id` AND have `location_status=ready_for_handover`. Violation raises `BINDER.HANDOVER_INVALID`. Creates a `BinderHandover` record after transitioning each binder.

### Bulk mark-ready (`BinderLifecycleService.mark_ready_for_handover_bulk`)
Iterates all client binders in `in_office` state whose last material period is ≤ the provided `(until_period_year, until_period_month)` cutoff. Skips binders not in `in_office`.

### Client onboarding
Cite: `backend/app/binders/services/client_onboarding_service.py`

When a new client is created, `create_initial_binder` opens a bare placeholder binder with `period_start=NULL`. Requires `office_client_number` to be set; if missing, raises `ValueError` (skipped with warning in some callers).

### Intake editing (`BinderIntakeEditService`)
Cite: `backend/app/binders/services/binder_intake_edit_service.py`

Exposed via `PATCH /api/v1/binders/{binder_id}/intakes/{intake_id}`. Supports partial edits to `BinderIntake` with field-level audit trail (`BinderIntakeEditLog`). Patchable intake fields: `received_at`, `received_by`, `notes`. Transfer patch (changing `client_record_id` or `binder_id`) validates that all FK-linked entities (`business_id`, `annual_report_id`, `vat_report_id`) belong to the target client/legal entity. The `binder_id` path parameter scopes the lookup — intakes belonging to a different binder return 404.

### Audit logging
Every lifecycle and capacity transition writes one `BinderLifecycleLog` row per changed field. `notes` carries the reason message. Initial state on creation logs `null → in_office` and `null → open`.

## Error codes

Cite: `backend/app/binders/services/messages.py`, `backend/app/binders/services/binder_lifecycle_service.py`

Registry: `docs/architecture/error-codes.md`.

| Code | When raised |
|------|-------------|
| `BINDER.NOT_FOUND` | Binder ID does not exist |
| `BINDER.NOT_INTAKE_ELIGIBLE` | Receive-material attempted when binder is not `in_office + open` |
| `BINDER.ALREADY_FULL` | mark_full on a binder already full |
| `BINDER.NOT_FULL` | reopen_capacity on a binder already open |
| `BINDER.CAPACITY_CHANGE_NOT_ALLOWED` | Capacity change attempted when `location_status != in_office` |
| `BINDER.INVALID_LOCATION_TRANSITION` | Unsupported location_status transition |
| `BINDER.NOT_READY_FOR_HANDOVER` | handover_to_client on a non-ready binder |
| `BINDER.ALREADY_HANDED_OVER` | Lifecycle mutation on an already-handed-over binder |
| `BINDER.CLIENT_LOCKED` | Intake attempted for a client whose all businesses are inactive |
| `BINDER.OFFICE_NUMBER_MISSING` | New binder creation attempted without `office_client_number` on client |
| `BINDER.OLD_PERIOD_NOTE_REQUIRED` | Old-period material inserted without an intake note |
| `BINDER.HANDOVER_INVALID` | Grouped handover: binder not belonging to client or not ready |
| `BINDER.CROSS_CLIENT` | Intake transfer: FK entity does not belong to target client |

## Known issues

Checked for all five recurring patterns from the authoring guide.

**F-011 — resolved**
`PATCH /api/v1/binders/{binder_id}/intakes/{intake_id}` added in `backend/app/binders/api/binders_history.py`. The endpoint scopes by `binder_id` before delegating to `BinderIntakeEditService.edit_intake`.

No IDOR found on lifecycle transitions: all update/delete paths load the binder by ID with `get_by_id_for_update` before mutating; grouped handover validates `binder.client_record_id == client_record_id` explicitly.

No unenforced invariants found in the state machine: code matches documented transition rules.

No stale computed-field source found: `period_start`/`period_end` are derived from structured material fields, not from any legacy column.

No off-format error codes found: all raised codes follow `BINDER.REASON` pattern.

No broken imports found in the binders module.

## Decisions (preserved)

From `backend/docs/backend/domains/binder_lifecycle_refactor_spec.md` (completed refactor, still true):

- **Two-field lifecycle model.** `location_status` and `capacity_status` are separate concepts. `open` is capacity only; intake eligibility is strictly `in_office + open`.
- **`BinderLifecycleService` as sole owner.** No router, repo, or external service mutates `location_status`/`capacity_status` directly. This is enforced by design, not just convention.
- **Available actions from backend.** `get_available_action_keys_for_state` in `BinderLifecycleService` is the source of truth. Frontend renders `available_actions`; backend enforces every transition.
- **Backend returns values only; no Hebrew labels.** UI labels live in frontend constants.
- **One binder per client at a time for intake.** All businesses share the same binder; material type is distinguished at `BinderIntakeMaterial` level.
- **Binder lifecycle is logistics-driven, not a strict reporting-period partition.** `period_start` is derived from the first inserted material and old-period material may still be routed into a newer in-office binder when no suitable older binder exists, with a required intake note.
- **A binder may span multiple reporting periods.** `period_start`/`period_end` are operational anchors for the physical binder, not a guarantee that all contained material belongs to one single accounting period.
- **Repeated arrivals stay as separate intake events.** Additional client deliveries create new `BinderIntake` rows rather than merging prior physical arrivals; duplicate material rows for the same type/period/business are acceptable for receipt tracking.
- **Bulk readiness is a preserved business concept.** Handover readiness can be decided across multiple binders up to a reporting-period cutoff, not only one binder at a time.
- **Post-handover binder is never reused.** New material after handover always opens a new binder.
- **`BinderLifecycleLog` replaces any old status log shape.** One row per changed field, not one row per transition event.

## Future / planned

- **Cross-client intake transfer UI.** The service and API endpoint exist; frontend form is not yet built.

## Historical notes

Pointer to `docs/archive/binders-legacy.md` for archived legacy lifecycle spec.
