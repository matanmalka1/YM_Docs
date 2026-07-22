## Scope
This file owns only:
- Canonical current-state documentation for the tasks domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Tasks

The tasks domain owns office-managed manual tasks. A task can be standalone, or it can be linked to a source record from another workflow domain so it can participate in the shared work queue. The domain itself is small: one `Task` model, CRUD/lifecycle endpoints, and source-link validation.

Last verified against code + backend/openapi.json: 2026-07-22.

## Endpoints

All endpoints require role `ADVISOR` or `SECRETARY` via the router-level dependency in `backend/app/tasks/api/routes.py`. All paths below exist in `backend/openapi.json`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/tasks | List non-deleted tasks with search, status/priority/assignee/source/due-date/`client_record_id` filters, allowlisted sorting, pagination, enriched client/assignee identity, and status summary |
| POST | /api/v1/tasks | Create a new manual task |
| POST | /api/v1/tasks/bulk-complete | Bulk-complete up to 100 tasks; partial success; requires `X-Idempotency-Key` |
| POST | /api/v1/tasks/bulk-assign | Bulk-assign (or unassign) up to 100 tasks; partial success; requires `X-Idempotency-Key` |
| GET | /api/v1/tasks/linkable-sources | List client-scoped open system work items that can be selected as a task source, including the existing linked-task count |
| GET | /api/v1/tasks/{task_id} | Fetch one non-deleted task by id |
| PATCH | /api/v1/tasks/{task_id} | Partially update a task that is still open |
| POST | /api/v1/tasks/{task_id}/complete | Mark an open task as done |
| POST | /api/v1/tasks/{task_id}/cancel | Mark an open task as canceled |
| DELETE | /api/v1/tasks/{task_id} | Soft-delete a task |

## Model & fields

Model: `Task` in `backend/app/tasks/models/task.py:29-69`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | auto-increment |
| `title` | `String(500)` | no | required in create schema (`backend/app/tasks/schemas/task.py:13-24`) |
| `description` | text | yes | |
| `status` | `TaskStatus` | no | default `open` (`backend/app/tasks/models/task.py:37-39`) |
| `priority` | `TaskPriority` | no | default `normal` (`backend/app/tasks/models/task.py:40-44`) |
| `due_date` | date | yes | |
| `assigned_to_user_id` | int FK -> `users.id` | yes | |
| `assigned_role` | `UserRole` | yes | stored in the shared `userrole` enum (`backend/app/tasks/models/task.py:47-49`) |
| `source_domain` | `String(100)` | yes | link target domain key |
| `source_id` | int | yes | link target primary key |
| `client_record_id` | int FK -> `client_records.id` | yes | direct client scope; `NULL` = global/system/unresolved legacy task; backfilled from source on migration `fc31c862b173` |
| `action_key` | `String(100)` | yes | optional UI/action hint payload key |
| `action_payload` | JSON | yes | optional action payload |
| `created_by_user_id` | int FK -> `users.id` | yes | set from authenticated creator in `create_task` |
| `completed_by_user_id` | int FK -> `users.id` | yes | set only on completion |
| `completed_at` | datetime | yes | set only on completion |
| `canceled_by_user_id` | int FK -> `users.id` | yes | set only on cancel |
| `canceled_at` | datetime | yes | set only on cancel |
| `created_at` | datetime | no | default `utcnow` |
| `updated_at` | datetime | no | default `utcnow`, auto-updated |
| `deleted_at` | datetime | yes | soft-delete marker |

List/detail responses enrich the persisted identifiers with `assigned_to_user_name`, `client_name`, and `office_client_number`. The detail response also enriches `created_by_user_name`, `completed_by_user_name`, and `canceled_by_user_name`. These are response-only fields resolved in batches; they are not stored on `Task`.

`GET /api/v1/tasks` returns a thin list row plus `summary: { total, open, done, canceled }`. The summary respects all current filters except `status`, so status cards can switch between lifecycle buckets without losing the other filter context.

Indexes: `status`, `priority`, `due_date`, `assigned_to_user_id`, `(source_domain, source_id)`, and `client_record_id` (`backend/app/tasks/models/task.py`).

## Enums / statuses

### TaskStatus (`backend/app/tasks/models/task.py:16-19`)

| Value |
|-------|
| `open` |
| `done` |
| `canceled` |

### TaskPriority (`backend/app/tasks/models/task.py:22-26`)

| Value |
|-------|
| `low` |
| `normal` |
| `high` |
| `urgent` |

### Assigned role (`backend/app/users/models/user.py:14-17`)

| Value |
|-------|
| `advisor` |
| `secretary` |

### Validated source domains

`Task.source_domain` is stored as a string, but `TaskService` validates it through `normalize_source_domain` before create/update. The shared enum defines `vat_work_item`, `annual_report`, `advance_payment`, `charge`, `binder`, and `task` (`backend/app/common/source_types.py:6-20`). In practice, only `vat_work_item`, `annual_report`, `advance_payment`, `charge`, and `binder` are resolvable link targets because `source_exists()` maps only those models (`backend/app/tasks/services/source_validator.py:14-29`).

These five domains also own the `client_record_id` backfill: when a task is linked to one of them, the task's `client_record_id` is resolved from the source record's own `client_record_id`. Tasks with `source_domain='task'` or soft-deleted source rows have `client_record_id=NULL` by design (`backend/alembic/versions/fc31c862b173_add_task_client_record_id.py`).

## Domain rules & invariants

- Create/update payloads validate `title` length, positive assignee/source ids, and enum values for `priority` and `assigned_role` through `TaskCreateRequest` / `TaskUpdateRequest` (`backend/app/tasks/schemas/task.py:13-36`).
- A task may be either standalone (`source_domain=None` and `source_id=None`) or linked. Partial links are rejected: create/update must provide both fields together, or clear both together (`backend/app/tasks/services/task_service.py:84-105,145-152`).
- A linked task must reference a supported source domain and an existing, non-soft-deleted source record. Unsupported source types raise `TASK.INVALID_SOURCE`; missing/deleted linked records raise `TASK.NOT_FOUND` (`backend/app/tasks/services/task_service.py:145-157`, `backend/app/tasks/services/source_validator.py:14-29`).
- New tasks are persisted through `TaskRepository.create()` with model defaults, so status starts as `open`, priority defaults to `normal`, and `created_by_user_id` is copied from the authenticated caller when present (`backend/app/tasks/services/task_service.py:28-43`, `backend/app/tasks/repositories/task_repository.py:48-57`, `backend/app/tasks/models/task.py:37-44`).
- `GET /api/v1/tasks` lists only tasks where `deleted_at IS NULL` and supports `search` over title/description; filters for `client_record_id`, `status`, `priority`, `assigned_to_user_id`, `assigned_role`, `source_domain`, `source_id`, `due_before`, and `due_after`; and allowlisted `sort_by` values `created_at`, `due_date`, `priority`, and `title` with `order=asc|desc`. Sorting is stable with task id as a tie-breaker (`backend/app/tasks/repositories/task_repository.py`).
- `GET /api/v1/tasks/linkable-sources` reuses the work-queue read model with the client pinned and `scope=system`. It deliberately keeps already-linked system items and returns `linked_tasks_count`, allowing the UI to warn instead of silently enforcing a one-task-per-source rule (`backend/app/tasks/services/task_service.py`).
- `get()` treats soft-deleted rows as not found. Every mutating operation (`update`, `complete`, `cancel`, `delete`) loads through `get()` first, so deleted tasks cannot be mutated (`backend/app/tasks/services/task_service.py:71-143`).
- Tasks in terminal states cannot be edited. `update()` rejects both `done` and `canceled` tasks with `TASK.CONFLICT` (`backend/app/tasks/services/task_service.py:71-82`).
- Completion and cancellation are one-way transitions from `open` only. Completing a canceled task, completing an already-done task, canceling a done task, or canceling an already-canceled task all raise `TASK.CONFLICT` (`backend/app/tasks/services/task_service.py:107-131`).
- `delete()` is a soft delete only: it sets `deleted_at` and `updated_at`; there is no restore endpoint in this domain (`backend/app/tasks/services/task_service.py:133-137`).

## Domain rules & invariants — client scoping

- `client_record_id` is populated automatically when a task is created with a supported source link. If the request also supplies `client_record_id` explicitly, the service validates the two are consistent (same client); a mismatch raises `TASK.CLIENT_SOURCE_MISMATCH` (`backend/app/tasks/services/task_service.py`).
- A task created without a source link but with an explicit `client_record_id` is validated: the client must exist, or `CLIENT_RECORD.NOT_FOUND` is raised.
- When a task's source is changed via PATCH, `client_record_id` is recomputed from the new source (source authoritative).
- When a task's source is cleared via PATCH (`source_domain=null, source_id=null`), `client_record_id` is preserved (the task becomes a manual client-scoped task). `client_record_id` is not directly patchable; scope only changes through source updates or at creation time.
- `GET /api/v1/tasks` accepts `client_record_id` as an optional filter query param (`TaskService.list(client_record_id=...)` → `TaskRepository.list_active(client_record_id=...)`), matching directly against `Task.client_record_id` (no source joins). Unlike the create/update path, this filter does not validate that the client exists: an unknown or nonexistent `client_record_id` simply yields an empty list, not `CLIENT_RECORD.NOT_FOUND`.

## Domain rules & invariants — bulk operations

- `POST /api/v1/tasks/bulk-complete` and `POST /api/v1/tasks/bulk-assign` accept 1–100 task IDs and require `X-Idempotency-Key`.
- Both return a partial-success response `{ succeeded: int[], failed: { task_id, code, message }[] }` in first-seen deduplicated order.
- Duplicate IDs in a single request are deduplicated before processing.
- Bulk complete: DONE tasks are idempotent successes (no mutation). CANCELED tasks fail with `TASK.CONFLICT`.
- Bulk assign: DONE or CANCELED tasks fail with `TASK.CONFLICT`. Tasks already assigned to the requested user are idempotent successes. Null `assignee_user_id` unassigns. If the requested assignee is not found or inactive, the entire request fails 404 (`TASK.INVALID_ASSIGNEE`) before any mutations occur.
- Missing task IDs fail per-item with `TASK.NOT_FOUND` (not a whole-request failure).

## Error codes

Registry: `docs/backend/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `TASK.NOT_FOUND` | Task id does not exist, task is soft-deleted, or a linked source record does not exist (`backend/app/tasks/services/task_service.py`) |
| `TASK.CONFLICT` | Attempt to edit a terminal task, complete a canceled/done task, cancel a done/canceled task, or bulk-operate on a terminal task (`backend/app/tasks/services/task_service.py`) |
| `TASK.INVALID_SOURCE` | Source link is partial or uses an unsupported source domain (`backend/app/tasks/services/task_service.py`) |
| `TASK.INVALID_ASSIGNEE` | Bulk assign target user not found or inactive (whole-request 404) (`backend/app/tasks/services/task_service.py`) |
| `TASK.CLIENT_SOURCE_MISMATCH` | Source record belongs to a different client than the explicitly provided `client_record_id` (`backend/app/tasks/services/task_service.py`) |
| `CLIENT_RECORD.NOT_FOUND` | Explicit `client_record_id` does not exist when creating a standalone task (`backend/app/tasks/services/task_service.py`) |

## Known issues

No open known issues.

## Resolved issues

- **Tasks-001** (2026-06-15): `task_service.py:_validate_client_exists` called `ClientRecordRepository` inline and raised `CLIENT.NOT_FOUND`. Replaced with `get_client_or_raise` → now raises `CLIENT_RECORD.NOT_FOUND`.
- **F-012** (2026-06-05): `GET /api/v1/tasks` accepted `assigned_role` and `source_domain` as plain strings, so invalid values bypassed validation and silently returned empty results. Fixed: route filters now use `UserRole` and `WorkQueueSourceType` (`backend/app/tasks/api/routes.py:32-33`), and the service accepts those typed values (`backend/app/tasks/services/task_service.py:51-52`).

## Decisions (preserved)

Still true from the canonical work-queue doc and verified against code:

1. **Tasks are the manual work items of the office queue.** The shared `WorkQueueSourceType` reserves `TASK` specifically for user-created tasks, distinct from computed system items (`docs/domains/work-queue.md`, `backend/app/common/source_types.py:6-12`, `backend/app/work_queue/services/task_items.py:31-61`).
2. **Open linked tasks merge into the source row; otherwise tasks stay standalone.** During work-queue assembly, an open task linked to a currently materialized source item is attached as `linked_tasks`; otherwise the task becomes its own `TASK` row, optionally enriched with source summary/warnings (`docs/domains/work-queue.md`, `backend/app/work_queue/services/work_queue_service.py:336-426`).

## Future / planned

No tasks-specific planned behavior could be verified from `backend/docs`. Do not treat any additional restore flow, extra statuses, or task-to-task linking as current behavior.

## Historical notes

No `backend/docs` file exists that is specific to the tasks domain, so this task did not create an archive file or replace any legacy file with a pointer. Historical context that mentions tasks still lives under the separate `work-queue` domain docs and remains owned by that domain.
