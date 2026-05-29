## Scope

This file owns only:
- Documentation of the work queue assembly flow as it exists in the code.

Source of truth: reference

# Flow: Work Queue Assembly

## 1. Trigger

`GET /work-queue` or `GET /work-queue/full`

## 2. Entry Point in Code

```
backend/app/work_queue/services/work_queue_service.py
  WorkQueueService.list_items()          # paginated
  WorkQueueService.list_items_with_total()   # with summary counts
    → _filtered_items()
      → _build_items()         # assemble system items from all domains
      → _apply_mode()          # active vs history task filter
      → apply_work_queue_filters()  # search, source_type, urgency, etc.
    → sort
    → paginate
```

## 3. Sequence

### Phase 1 — System item assembly (`_build_items`)

All system items are derived from live DB data. No `WorkQueueItem` rows are stored.

**Client-level items** (skipped when `business_id` filter is set):
- `vat_work_item_items(ctx, client_record_id)` — non-FILED, non-CANCELED VatWorkItems with upcoming/overdue deadlines.
- `annual_report_items(ctx, client_record_id)` — non-final AnnualReports with deadlines.
- `advance_payment_items(ctx, client_record_id)` — PENDING/PARTIAL AdvancePayments within `UPCOMING_WINDOW_DAYS` of due_date.

**All items** (available with or without business_id filter):
- `charge_items(ctx, client_record_id, business_id)` — ISSUED charges 30+ days old (always OVERDUE).

**Unfiltered only** (only when no client_record_id or business_id filter):
- `binder_items(ctx)` — stale binders meeting specific criteria.

Each system item: `available_actions = source_actions(source_type, source_id)`.

### Phase 2 — Task merge (`_merge_tasks`)

Skipped if `business_id` filter is set (tasks are not scoped to a specific business).

1. Load all tasks via `TaskRepository.list_for_work_queue(include_history)`.
2. Build `linked_keys`: set of `(source_type, source_id)` pairs from tasks that have a `source_domain`.
3. `load_source_states(db, linked_keys)` — batch-load source state (label, route, final/missing/deleted) for all linked task sources.
4. For each task:
   - If task has a recognized `source_domain` + `source_id`:
     - If that source_id already has a system item and task is OPEN: **merge** task onto system item via `_attach_task()`.
     - Otherwise: add as standalone `WorkQueueItem` (task row).
   - If task has no source: add as standalone.
   - Client filter applied: skip if `client_record_id` set and task belongs to a different client.

### Phase 3 — Filters and sort

`apply_work_queue_filters(items, filters)` — filter by:
- `search`: text match across title, description, client_name, office_client_number, business_id, type_label, status_label, source_type, period, priority, assigned_role, linked tasks.
- `source_type`: exact match.
- `urgency`: exact match.
- `task_status`: match on item or any linked task.
- `linked`: filter to items with/without linked tasks.
- `scope`: MANUAL (tasks only) or SYSTEM (non-task items only).

Sort key: `urgency → due_date → item_kind (task vs system) → task_status → priority → title → source_type → source_id → id`.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `work_queue` | Owns the service and assembly |
| `vat_reports` | VatWorkItem (read) |
| `annual_reports` | AnnualReport (read) |
| `advance_payments` | AdvancePayment (read) |
| `charge` | Charge (read) |
| `binders` | Binder (read, stale binders) |
| `tasks` | Task (read, merging and standalone) |
| `clients` | ClientRecord identity enrichment, active-client scope |

## 5. Side Effects

None. Read-only.

## 6. Transaction Boundaries

Read-only. All queries in one DB session. No writes.

## 7. Idempotency

N/A — read-only. Each call returns the current live snapshot.

## 8. Locks / Concurrency

None.

## 9. Preconditions

None.

## 10. Blockers / Validation Failures

None at read time.

## 11. Derived State

Nothing is stored. All items are computed at query time.

| Source Type | Appears When | Always Overdue? |
|-------------|-------------|-----------------|
| VAT_WORK_ITEM | Non-FILED, non-CANCELED, deadline approaching/past | No — urgency from due_date |
| ANNUAL_REPORT | Non-final, deadline set | No — urgency from due_date |
| ADVANCE_PAYMENT | PENDING or PARTIAL, due within UPCOMING_WINDOW_DAYS | No — urgency from due_date |
| CHARGE | ISSUED, issued 30+ days ago | Yes — always OVERDUE |
| BINDER | Stale in-office binders | See binder_items logic |
| TASK | All open tasks (or history if requested) | No — urgency from task.due_date |

Urgency levels: `OVERDUE`, `APPROACHING`, `IMPORTANT`, `UPCOMING` — computed from `due_date` vs `israel_today()`.

## 12. N+1 Risk

**Mitigated:**
- `ctx.preload_client_identities(...)` batch-loads client names for all matched items per sub-list.
- `load_source_states(db, linked_keys)` batch-loads source state for all task-linked sources in one query.

**Potential risk:**
- If a large number of tasks link to different sources, `load_source_states` may issue a query with a large `IN` clause.
- No documented query limit on the number of tasks returned from `task_repo.list_for_work_queue()`.

## 13. Task Merging Rules

- Only OPEN tasks are merged onto system items.
- Closed/canceled tasks appear separately if `include_task_history=True`.
- If multiple tasks link to the same system item: all are merged; a warning is added to the item.
- Task urgency can upgrade the system item's urgency (if task.due_date is sooner).
- If a task's source is missing/deleted from DB: warning is added to the standalone task item.

## 14. Tests

- `tests/work_queue/test_work_queue_service.py`
- `tests/work_queue/test_work_queue_api.py`
- `tests/work_queue/test_task_work_queue.py`

**GAP**: `binder_items` was not read in detail — no documentation of the exact staleness criteria for binder work queue items.

## 15. Documentation Target

- `docs/domains/work-queue.md` — source types, assembly, task merging, filters
- `docs/domains/charge.md` — charge items appearance in work queue
