## Scope

This file owns only:
- Documentation of how charge items appear in the work queue.

Source of truth: reference

# Flow: Charge → Work Queue Derived Item

## 1. Trigger

Any work queue read: `GET /work-queue` or `GET /work-queue/full`.
No DB writes. Pure read-time derivation.

## 2. Entry Point in Code

```
backend/app/work_queue/services/billing_items.py
  charge_items(ctx, client_record_id, business_id)
```

Called from:
```
backend/app/work_queue/services/work_queue_service.py
  WorkQueueService._build_items()
    → charge_items(ctx, client_record_id, business_id)
```

## 3. Sequence

1. Compute `threshold = ctx.today - timedelta(days=UNPAID_CHARGE_TASK_THRESHOLD_DAYS)`.
   - `UNPAID_CHARGE_TASK_THRESHOLD_DAYS = 30` (defined in `backend/app/charges/services/constants.py`).

2. Query `Charge` table:
   ```
   WHERE deleted_at IS NULL
     AND status = 'issued'
     AND issued_at IS NOT NULL
     AND issued_at <= threshold
     AND client_record_id IN (active clients only)   -- scope_to_active_clients_stmt
     [AND client_record_id = ?]   -- if client filter provided
     [AND business_id = ?]        -- if business filter provided
   ```

3. `ctx.preload_client_identities(charge.client_record_id for charge in charges)` — batch-loads client identity for all matched charges.

4. For each charge: build `WorkQueueItem`:
   - `source_type`: `CHARGE`
   - `source_id`: `charge.id`
   - `title`: `"חיוב לא שולם"`
   - `due_date`: `charge.issued_at.date() + timedelta(days=30)` — computed, not stored
   - `urgency`: always `WorkQueueUrgency.OVERDUE` (items appear only after threshold passed)
   - `status_label`: `charge.status.value`
   - `metadata`: `charge_metadata(charge, due_date)`
   - `business_id`: `charge.business_id`

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `charge` | Source of truth for charge data |
| `clients` | Active client scope filter; client identity enrichment |
| `work_queue` | Builds WorkQueueItem (not stored) |

## 5. Side Effects

None. Read-only.

## 6. Transaction Boundaries

Read-only. No writes.

## 7. Idempotency

N/A — read-only.

## 8. Locks / Concurrency

None.

## 9. Preconditions

None. Always returns current snapshot.

## 10. Blockers / Validation Failures

None at read time. Items appear as long as the charge meets the criteria.

## 11. Derived State

No `WorkQueueItem` row is stored in the DB.
All fields are computed at query time:
- `due_date = charge.issued_at + 30 days` (not stored on Charge)
- `urgency = OVERDUE` (always, because items appear only after threshold)

A charge stops appearing in the work queue when:
- `status` changes away from `ISSUED` (paid, canceled, deleted), or
- The charge is soft-deleted.

There is no explicit "dismiss" or "snooze" for charge work queue items.

## 12. N+1 Risk

`ctx.preload_client_identities(...)` batch-loads all client identities in one query — no N+1.

## 13. Tests

- `tests/work_queue/test_work_queue_service.py`
- `tests/work_queue/test_work_queue_api.py`
- `tests/actions/test_charge_actions.py`
- `tests/regression/test_core_regressions_binders_charges_notifications.py`

## 14. Documentation Target

- `docs/domains/charges.md` — work queue derivation, 30-day threshold
- `docs/domains/work-queue.md` — charge items, source types, urgency
