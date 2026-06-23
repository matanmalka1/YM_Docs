## Scope
This file owns only:
- Canonical current-state documentation for the work-queue domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Work Queue

The work-queue domain is a read-only aggregator that turns work from multiple backend domains into one office-facing queue. It does not own a database table or lifecycle of its own; instead it materializes response rows from source records such as VAT work items, annual reports, advance payments, unpaid charges, stale binders, and manual tasks.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All endpoints require role `ADVISOR` or `SECRETARY` via `require_role(...)` in `backend/app/work_queue/api/routes.py:49-53`. The path below exists in `backend/openapi.json:12156-12372`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/work-queue | Return the filtered queue page plus `total` and `summary` for the full filtered result set |

## Model & fields

This module defines no ORM models or database table of its own. Current state lives in response schemas under `backend/app/work_queue/schemas/work_queue.py:13-91`, and manual queue rows are derived from `Task` records in `backend/app/tasks/models/task.py:29-69`.

### Response item shape

`WorkQueueItem` is the canonical row schema (`backend/app/work_queue/schemas/work_queue.py:53-72`):

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `id` | string | no | Stable synthetic key, always `{source_type}:{source_id}` |
| `source_type` | `WorkQueueSourceType` | no | Source-domain discriminator |
| `source_id` | int | no | Source record primary key |
| `title` | string | no | Queue label shown to users |
| `description` | string | yes | Present mainly for task rows |
| `type_label` | string | yes | Hebrew source label from `SOURCE_TYPE_LABELS` |
| `status_label` | string | yes | Hebrew status label derived from source status |
| `due_date` | date | yes | Queue due date used for urgency/sorting |
| `urgency` | `WorkQueueUrgency` | no | Computed queue urgency |
| `client_record_id` | int | yes | Attached directly or later via source lookup |
| `client_name` | string | yes | Lazy-resolved from client identity map |
| `office_client_number` | int | yes | Lazy-resolved office number |
| `business_id` | int | yes | Populated only for business-scoped source rows such as charges |
| `source_summary` | object | yes | Link target summary for standalone linked tasks |
| `linked_tasks` | list | no | Open linked tasks merged into a system row |
| `linked_tasks_count` | int | no | Count of merged open tasks |
| `warnings` | list | no | Queue warnings such as missing or final linked source |
| `available_actions` | list | no | UI action descriptors for source/task actions |
| `metadata` | dict | yes | Source-specific payload such as period, tax year, charge amount, or task assignment |

### Persisted source for manual rows

Standalone/manual queue rows come from `Task` and serialize these persisted fields into `metadata`: `status`, `priority`, `description`, `assigned_to_user_id`, `assigned_role`, `action_key`, `action_payload`, `source_domain`, and `source_id` (`backend/app/work_queue/services/task_items.py:31-60`, `backend/app/tasks/models/task.py:32-61`).

### Summary shape

`WorkQueueSummary` contains `total`, `manual_tasks`, `linked`, `unlinked`, urgency buckets, `by_source_type`, and `by_task_status` (`backend/app/work_queue/schemas/work_queue.py:75-91`).

## Enums / statuses

### WorkQueueSourceType

Exact values from `backend/app/common/source_types.py:6-12`:

| Value |
|-------|
| `vat_work_item` |
| `annual_report` |
| `advance_payment` |
| `charge` |
| `binder` |
| `task` |

### WorkQueueUrgency

Exact values from `backend/app/work_queue/schemas/work_queue.py:13-18`:

| Value |
|-------|
| `overdue` |
| `approaching` |
| `important` |
| `upcoming` |

### WorkQueueLinkedFilter

Exact values from `backend/app/work_queue/schemas/work_queue.py:20-22`:

| Value |
|-------|
| `linked` |
| `unlinked` |

### WorkQueueScope

Exact values from `backend/app/work_queue/schemas/work_queue.py:25-27`:

| Value |
|-------|
| `system` |
| `manual` |

### Status labels

The queue does not define its own persisted lifecycle statuses. Instead it maps source-domain statuses into Hebrew display labels for VAT, annual reports, advance payments, charges, binders, and tasks via `STATUS_LABELS` (`backend/app/work_queue/services/common.py:32-71`).

## Domain rules & invariants

- `GET /api/v1/work-queue` accepts filters for `client_record_id`, `business_id`, `exclude_source_types`, `search`, `source_type`, `urgency`, `task_status`, `linked`, `scope`, `page`, and `page_size`; all are part of the published OpenAPI contract (`backend/app/work_queue/api/routes.py`, `backend/openapi.json`). The queue is forward-looking only: completed/canceled tasks never appear — review them on the dedicated tasks page (`GET /api/v1/tasks`).
- `GET /api/v1/work-queue` returns one queue page and one summary for the supplied filters. It can summarize a single `client_record_id`, but it does not expose batched per-client counts for all sidebar clients (`backend/app/work_queue/api/routes.py:45-67`, `backend/app/work_queue/services/work_queue_service.py:180-252`).
- The queue is built fully in memory on every request: `_build_items()` gathers candidate rows (only open tasks are ever fetched), `apply_work_queue_filters()` applies Python-side filters, and only then does pagination slice the page (`backend/app/work_queue/services/work_queue_service.py`).
- Summary is computed over the full filtered set before pagination, not over the current page (`backend/app/work_queue/services/work_queue_service.py:246-252`).
- System rows come from source builders only. `vat_work_item`, `annual_report`, and `advance_payment` are client-level obligations and are skipped entirely when `business_id` is supplied; `charge` rows can be narrowed by both client and business; `binder` rows appear only on the completely unscoped queue (`backend/app/work_queue/services/work_queue_service.py:283-304`).
- VAT rows come from overdue/unfiled `VatWorkItem` records, use `due_date_effective`, and raise urgency from that effective due date (`backend/app/work_queue/services/tax_items.py:26-62`).
- Annual-report rows include reports with a non-null `filing_deadline` up to 21 days ahead, excluding `submitted`, `closed`, and `canceled` statuses (`backend/app/work_queue/services/tax_items.py:19-24,65-95`).
- Advance-payment rows include non-deleted `pending` and `partial` payments due within the next 21 days (`backend/app/work_queue/services/billing_items.py:26-55`).
- Charge rows are derived only from issued unpaid charges whose `issued_at` is at least `UNPAID_CHARGE_TASK_THRESHOLD_DAYS` old; the queue due date is computed as `issued_at.date() + threshold`, and queue urgency is forced to `overdue` because the threshold has already passed before the item appears (`backend/app/work_queue/services/billing_items.py:58-92`).
- Binder rows are derived from binders that have stayed `ready_for_handover` for more than 30 days and are always marked `overdue` (`backend/app/work_queue/services/binder_items.py:11-39`).
- Open linked tasks merge into the matching system row as `linked_tasks`. Tasks that are closed, unlinked, linked to a source that is not currently materialized, or linked to an unknown source type remain standalone `task` rows (`backend/app/work_queue/services/work_queue_service.py:336-426`).
- Standalone linked task rows can carry warnings: `source_missing`, `source_final`, or `source_unknown`. System rows with more than one linked open task get `multiple_linked_tasks` (`backend/app/work_queue/services/work_queue_service.py:388-441`).
- Linked tasks can only escalate a system row's urgency. If a linked task is more urgent than the underlying source row, the queue row adopts the task due date/urgency; it is never downgraded (`backend/app/work_queue/services/work_queue_service.py:470-475`).
- Sorting is deterministic: urgency bucket, due date, item kind (`task` rows before system rows), task status rank, task priority rank, title, source type, source id, then synthetic row id (`backend/app/work_queue/services/work_queue_service.py:36-47,493-513`).
- Client identity is resolved lazily in batch through `WorkQueueContext`, so builders register visible client ids first and then hydrate `client_name` / `office_client_number` from `ClientIdentityRepository` instead of issuing per-row lookups (`backend/app/work_queue/services/common.py:100-201`).

## Error codes

Registry: `docs/backend/error-codes.md`.

This module raises no `WORK_QUEUE.*` domain error codes. The route is read-only and relies on:

- auth/permission enforcement from `require_role(...)` (`backend/app/work_queue/api/routes.py:49-53`);
- FastAPI request validation for invalid query parameters (`backend/openapi.json:12168-12372`);
- one internal consistency exception: `_vat_due_date()` raises a raw `ValueError` if a `VatWorkItem` reaches the queue without `due_date_effective` (`backend/app/work_queue/services/tax_items.py:26-30`).

## Known issues

No open known issues.

Previously tracked work-queue findings are resolved:

- **F-020**: Advance-payment queue rows now filter by `due_date_effective` and render the effective due date, falling back to legacy `due_date` only when no effective snapshot exists (`backend/app/work_queue/services/billing_items.py:29-54`).
- **F-021**: `business_id` scope no longer disables task merging. Open linked tasks can still attach to matching materialized system rows; only standalone task rows are skipped when a business filter is active (`backend/app/work_queue/services/work_queue_service.py:336-427`).

## Decisions (preserved)

Still-true decisions carried forward from `docs/archive/work-queue-legacy.md`, verified against current code:

1. **The work queue is computed, not persisted.** There is no `work_queue` table; every request rebuilds the queue from source domains and task data (`docs/archive/work-queue-legacy.md:32-33,63`, `backend/app/work_queue/services/work_queue_service.py:180-252`).
2. **`task` is the only manual source type; the others are derived system rows.** The shared source enum still separates user-created tasks from computed queue items (`docs/archive/work-queue-legacy.md:32-39`, `backend/app/common/source_types.py:6-12`).
3. **Open linked tasks merge into their source row; otherwise they stay standalone.** This remains the core queue/task integration rule (`docs/archive/work-queue-legacy.md:41-48`, `backend/app/work_queue/services/work_queue_service.py:336-426`).
4. **Summary reflects the filtered set before pagination.** The response still returns page items plus a summary over all filtered rows (`docs/archive/work-queue-legacy.md:21-26`, `backend/app/work_queue/services/work_queue_service.py:247-252`).

## Future / planned

- `backend/docs/domain_decisions_v3.md` still marks removal of legacy `AdvancePayment.due_date` as future work after consumers are audited (`backend/docs/domain_decisions_v3.md:392-395`). Treat the current column as still present, but not as the long-term source of truth.
- The same decision log keeps an explicit due-date override endpoint for tax-calendar-driven deadlines as `Future / planned` only if product later needs it, with reason/permission/terminal-state guards (`backend/docs/domain_decisions_v3.md:392-395`).
- Batched work-queue counts for the client sidebar are not implemented. The existing endpoint can compute a filtered summary for one client at a time, but there is no navigation-specific response keyed by multiple `client_record_id` values (`backend/app/work_queue/api/routes.py:45-67`, `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:129-132`).

## Historical notes

Archived original: `docs/archive/work-queue-legacy.md`.
