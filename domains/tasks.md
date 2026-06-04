## Scope
This file owns only:
- Canonical current-state documentation for the tasks domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Tasks

The tasks domain owns office-managed manual tasks. A task can be standalone, or it can be linked to a source record from another workflow domain so it can participate in the shared work queue. The domain itself is small: one `Task` model, CRUD/lifecycle endpoints, and source-link validation.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All endpoints require role `ADVISOR` or `SECRETARY` via the router-level dependency in `backend/app/tasks/api/routes.py:18-22`. All paths below exist in `backend/openapi.json:11687-12155`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/tasks | List non-deleted tasks with status/priority/assignee/source/due-date filters and pagination |
| POST | /api/v1/tasks | Create a new manual task |
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

Indexes: `status`, `priority`, `due_date`, `assigned_to_user_id`, and `(source_domain, source_id)` (`backend/app/tasks/models/task.py:63-68`).

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

## Domain rules & invariants

- Create/update payloads validate `title` length, positive assignee/source ids, and enum values for `priority` and `assigned_role` through `TaskCreateRequest` / `TaskUpdateRequest` (`backend/app/tasks/schemas/task.py:13-36`).
- A task may be either standalone (`source_domain=None` and `source_id=None`) or linked. Partial links are rejected: create/update must provide both fields together, or clear both together (`backend/app/tasks/services/task_service.py:84-105,145-152`).
- A linked task must reference a supported source domain and an existing, non-soft-deleted source record. Unsupported source types raise `TASK.INVALID_SOURCE`; missing/deleted linked records raise `TASK.NOT_FOUND` (`backend/app/tasks/services/task_service.py:145-157`, `backend/app/tasks/services/source_validator.py:14-29`).
- New tasks are persisted through `TaskRepository.create()` with model defaults, so status starts as `open`, priority defaults to `normal`, and `created_by_user_id` is copied from the authenticated caller when present (`backend/app/tasks/services/task_service.py:28-43`, `backend/app/tasks/repositories/task_repository.py:48-57`, `backend/app/tasks/models/task.py:37-44`).
- `GET /api/v1/tasks` lists only tasks where `deleted_at IS NULL`, ordered by `created_at DESC`, and supports filters for `status`, `priority`, `assigned_to_user_id`, `assigned_role`, `source_domain`, `source_id`, `due_before`, and `due_after` (`backend/app/tasks/repositories/task_repository.py:59-98`).
- `get()` treats soft-deleted rows as not found. Every mutating operation (`update`, `complete`, `cancel`, `delete`) loads through `get()` first, so deleted tasks cannot be mutated (`backend/app/tasks/services/task_service.py:71-143`).
- Tasks in terminal states cannot be edited. `update()` rejects both `done` and `canceled` tasks with `TASK.CONFLICT` (`backend/app/tasks/services/task_service.py:71-82`).
- Completion and cancellation are one-way transitions from `open` only. Completing a canceled task, completing an already-done task, canceling a done task, or canceling an already-canceled task all raise `TASK.CONFLICT` (`backend/app/tasks/services/task_service.py:107-131`).
- `delete()` is a soft delete only: it sets `deleted_at` and `updated_at`; there is no restore endpoint in this domain (`backend/app/tasks/services/task_service.py:133-137`).

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `TASK.NOT_FOUND` | Task id does not exist, task is soft-deleted, or a linked source record does not exist (`backend/app/tasks/services/task_service.py:140-157`) |
| `TASK.CONFLICT` | Attempt to edit a terminal task, complete a canceled/done task, or cancel a done/canceled task (`backend/app/tasks/services/task_service.py:73-74,109-112,122-125`) |
| `TASK.INVALID_SOURCE` | Source link is partial or uses an unsupported source domain (`backend/app/tasks/services/task_service.py:87-99,148-155`) |

## Known issues


## Decisions (preserved)

Still true from the historical work-queue spec in `backend/docs/backend/domains/work-queue.md`, verified against code:

1. **Tasks are the manual work items of the office queue.** The shared `WorkQueueSourceType` reserves `TASK` specifically for user-created tasks, distinct from computed system items (`backend/docs/backend/domains/work-queue.md`, `backend/app/common/source_types.py:6-12`, `backend/app/work_queue/services/task_items.py:31-61`).
2. **Open linked tasks merge into the source row; otherwise tasks stay standalone.** During work-queue assembly, an open task linked to a currently materialized source item is attached as `linked_tasks`; otherwise the task becomes its own `TASK` row, optionally enriched with source summary/warnings (`backend/docs/backend/domains/work-queue.md`, `backend/app/work_queue/services/work_queue_service.py:336-426`).

## Future / planned

No tasks-specific planned behavior could be verified from `backend/docs`. Do not treat any additional restore flow, extra statuses, or task-to-task linking as current behavior.

## Historical notes

No `backend/docs` file exists that is specific to the tasks domain, so this task did not create an archive file or replace any legacy file with a pointer. Historical context that mentions tasks still lives under the separate `work-queue` domain docs and remains owned by that domain.
