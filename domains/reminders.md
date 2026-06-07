## Scope
This file owns only:
- Canonical current-state documentation for the reminders domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Reminders

The reminders domain stores scheduled reminder rows that may later create tasks, send notifications, or do both. In the current codebase it exposes only create/list/get/cancel APIs plus a background-executor service stub; scheduling metadata is persisted, but actual execution is not implemented.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

Router prefix is `/reminders` under `/api/v1`, so the effective paths below come from `backend/app/reminders/api/routers.py:12-20`. All four paths exist in `backend/openapi.json:7690-7892`.

Auth:
- Router-level access allows `ADVISOR` and `SECRETARY` (`backend/app/reminders/api/routers.py:12-16`).
- `POST /api/v1/reminders/` adds a route-level `ADVISOR` requirement, so create is advisor-only (`backend/app/reminders/api/routes_create.py:13-18`).

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/reminders/ | List reminders by status with pagination; defaults to scheduled reminders when `status` is omitted |
| POST | /api/v1/reminders/ | Create a reminder row from the request payload |
| GET | /api/v1/reminders/{reminder_id} | Fetch one reminder by id |
| POST | /api/v1/reminders/{reminder_id}/cancel | Cancel a scheduled reminder |

## Model & fields

Model: `Reminder` in `backend/app/reminders/models/reminder.py:28-52`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | autoincrement |
| `fire_at` | datetime | no | indexed |
| `status` | `ReminderStatus` | no | defaults to `scheduled`; indexed |
| `action_type` | `ReminderActionType` | no | required execution mode |
| `source_domain` | `String(100)` | yes | optional source-domain key |
| `source_id` | int | yes | optional source record id |
| `target_task_id` | int | yes | optional linked task id |
| `notification_template_key` | `String(100)` | yes | optional notification trigger/template key |
| `payload` | JSON | yes | arbitrary reminder payload |
| `created_by_user_id` | int | yes | copied from authenticated creator on create |
| `fired_at` | datetime | yes | set when executor attempts to process the reminder |
| `failure_reason` | text | yes | failure message written by the executor stub |
| `created_at` | datetime | no | defaults to `utcnow` |
| `updated_at` | datetime | no | defaults to `utcnow`, auto-updated on change |

Index:
- Composite index `idx_reminders_status_fire_at` on (`status`, `fire_at`) supports due-scheduled scans (`backend/app/reminders/models/reminder.py:52`).

Request/response schema notes:
- `ReminderCreateRequest` requires only `fire_at` and `action_type`; all linkage/template fields are optional (`backend/app/reminders/schemas/reminders.py:9-16`, `backend/openapi.json:22109-22188`).
- `ReminderResponse` adds derived `client_record_id`, `client_name`, and `office_client_number`; those are not stored columns on `reminders` (`backend/app/reminders/schemas/reminders.py:19-38`, `backend/openapi.json:22190-22362`).

## Enums / statuses

### ReminderActionType (`backend/app/reminders/models/reminder.py:15-18`, `backend/openapi.json:22100-22107`)

| Value | Meaning in current code |
|-------|-------------------------|
| `CREATE_TASK` | Intended future execution mode; currently persisted only |
| `SEND_NOTIFICATION` | Intended future execution mode; currently persisted only |
| `CREATE_TASK_AND_NOTIFY` | Intended future execution mode; currently persisted only |

### ReminderStatus (`backend/app/reminders/models/reminder.py:21-25`, `backend/openapi.json:22364-22372`)

| Value | Meaning |
|-------|---------|
| `scheduled` | Default persisted state and the list default when no `status` filter is provided |
| `fired` | Reserved success state; current executor never sets it |
| `canceled` | Terminal state written by the cancel flow |
| `failed` | Terminal state written by the current executor stub |

### Recognized source-domain values for client enrichment

`Reminder.source_domain` is stored as a free-form string, but response enrichment recognizes only the shared work-queue source types because `normalize_source_domain()` maps to `WorkQueueSourceType` (`backend/app/common/source_types.py:6-21`):

| Value |
|-------|
| `vat_work_item` |
| `annual_report` |
| `advance_payment` |
| `charge` |
| `binder` |
| `task` |

## Domain rules & invariants

- Create is thin persistence only: `create_from_request()` forwards request fields directly to `ReminderRepository.create()` and stamps `created_by_user_id` from the authenticated user. The service does not validate `source_domain`, `source_id`, `target_task_id`, or `notification_template_key` beyond schema shape checks (`backend/app/reminders/services/reminder_service.py:22-34`, `backend/app/reminders/repositories/reminder_repository.py:21-46`).
- `GET /api/v1/reminders/` defaults to `scheduled` reminders. If `status` is omitted, `_parse_status()` returns `ReminderStatus.SCHEDULED`; any other value must exactly match one of the enum values or the service raises `REMINDER.INVALID_STATUS` (`backend/app/reminders/services/reminder_service.py:36-42,150-159`).
- Listing is status-scoped, ordered by `fire_at ASC, id ASC`, and paginated through the shared repository pagination helper (`backend/app/reminders/repositories/reminder_repository.py:60-71`).
- Cancel is allowed only for reminders still in `scheduled`. Missing ids raise `REMINDER.NOT_FOUND`; any non-scheduled status raises `REMINDER.INVALID_STATUS`; successful cancellation writes `canceled` via `update_status()` (`backend/app/reminders/services/reminder_service.py:47-56`).
- Response client enrichment is derived, not stored. Resolution order is:
  1. `payload.client_record_id` if present and parseable as int.
  2. `source_domain` + `source_id` if the source domain normalizes to a known `WorkQueueSourceType`.
  3. The linked task's own `source_domain` + `source_id` when `target_task_id` points to a non-deleted task.
  4. Otherwise `client_record_id`, `client_name`, and `office_client_number` remain `null`.
  Cite: `backend/app/reminders/services/reminder_service.py:61-148`, `backend/app/tasks/repositories/task_repository.py:106-110`, `backend/app/work_queue/services/source_lookup.py:59-167`.
- Due-reminder scanning considers only rows where `status == scheduled` and `fire_at <= now`, ordered by earliest `fire_at` first, limited to 100 by default (`backend/app/reminders/repositories/reminder_repository.py:76-86`, `backend/app/reminders/services/reminder_executor_service.py:29-43`).
- The current executor never performs the requested action. `_execute()` always sets `status=failed`, stamps `fired_at`, writes a fixed unsupported-operation message into `failure_reason`, and returns `False` for every action type (`backend/app/reminders/services/reminder_executor_service.py:45-60`).

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `REMINDER.NOT_FOUND` | Cancel is requested for a reminder id that does not exist (`backend/app/reminders/services/reminder_service.py:47-50`) |
| `REMINDER.INVALID_STATUS` | Cancel is requested for a non-`scheduled` reminder, or the list `status` query is not one of the enum values (`backend/app/reminders/services/reminder_service.py:51-55,154-158`) |

## Known issues

1. **F-015 — Reminder execution is effectively non-functional.** The domain exposes reminder creation and stores `CREATE_TASK`, `SEND_NOTIFICATION`, and `CREATE_TASK_AND_NOTIFY`, but the only executor path always marks due reminders as `failed` with "not implemented" reasons and never writes `fired`. Locations: `backend/app/reminders/services/reminder_executor_service.py:45-60`. This violates the domain expectation that scheduled reminders can actually execute their declared action type. Suggested fix: implement real action dispatch per `ReminderActionType` and transition successful executions to `fired`.
2. **F-016 — No background job runs the reminder executor.** Application startup schedules only signature-request expiry; nothing invokes `ReminderExecutorService.fire_due()` on startup or on an interval. Locations: `backend/app/core/background_jobs.py:19-82`, `backend/app/lifespan.py:16-23`. This violates the operational expectation that due scheduled reminders are processed automatically. Suggested fix: add a dedicated reminder runner to the background-jobs module and register it in lifespan.

## Decisions (preserved)

Still-true decisions carried forward from the historical domain-model review and confirmed in code:

1. **Reminder rows are generic scheduling records, not client-owned workflow rows.** The table persists source metadata (`source_domain`, `source_id`, `target_task_id`, payload) but has no dedicated `client_record_id` or `business_id` column; client identity in API responses is derived on read (`backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md:120-128`, `backend/app/reminders/models/reminder.py:39-46`, `backend/app/reminders/services/reminder_service.py:61-148`).
2. **Execution support is intentionally incomplete in the current system state.** The historical review said reminder execution is unimplemented and unscheduled, and current code still matches that assessment (`backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md:83-93`, `backend/app/reminders/services/reminder_executor_service.py:29-60`, `backend/app/lifespan.py:19-23`).

## Future / planned

- Actual reminder execution for `CREATE_TASK`, `SEND_NOTIFICATION`, and `CREATE_TASK_AND_NOTIFY` remains planned only. Current code stores these action types but does not execute them (`backend/app/reminders/services/reminder_executor_service.py:55-60`).
- Automatic processing of due reminders is planned only. No scheduled background job currently calls `ReminderExecutorService.fire_due()` (`backend/app/core/background_jobs.py:61-82`, `backend/app/lifespan.py:16-23`).
- Historical analysis noted that reminders are not explicitly client-owned and cannot be bulk-canceled by client lifecycle. That capability is still absent; do not assume close/freeze/soft-delete flows automatically cancel reminders (`backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md:120-128`, `backend/app/reminders/models/reminder.py:39-46`, `backend/app/reminders/repositories/reminder_repository.py:48-86`).

## Historical notes

No dedicated `backend/docs` file exists that is owned only by the reminders domain, so this task did not create an archive file or replace any legacy file with a pointer. The only reminders-specific historical material found was embedded in shared cross-domain notes such as `backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md`, which were intentionally left untouched to avoid cross-domain edits.
