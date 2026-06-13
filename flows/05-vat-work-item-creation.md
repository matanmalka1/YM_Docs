## Scope

This file owns only:
- Documentation of the VAT work item creation flow as it exists in the code.

Source of truth: reference

# Flow: VAT Work Item Creation

## 1. Trigger

Two paths:
- **Manual**: `POST /vat/work-items` API endpoint.
- **Automatic**: called from `ClientOnboardingOrchestrator._sync_vat_work_items()` during client creation (see flow 01).

## 2. Entry Point in Code

```
backend/app/vat/services/intake.py
  create_work_item(work_item_repo, db, *, client_record_id, period, created_by, ...)
    → VatClientContextService.get_active_client_and_entity()
    → resolve_effective_vat_type()
    → _validate_period_for_vat_type()
    → work_item_repo.get_by_client_record_period()       # conflict check
    → TaxCalendarMaterializationService.ensure_periodic_entry()
    → work_item_repo.create()
    → materializer.link_vat_work_item()
    → db.flush()
    → work_item_repo.append_audit()
```

## 3. Sequence

1. `VatClientContextService.get_active_client_and_entity(client_record_id)`:
   - Load `ClientRecord` and associated `LegalEntity`.
   - Raises `NotFoundError` if not found.

2. Guard: `client_record.status == CLOSED` → `AppError("VAT.CLIENT_CLOSED")`.
3. Guard: `client_record.status == FROZEN` → `AppError("VAT.CLIENT_FROZEN")`.

4. `resolve_effective_vat_type(legal_entity)` — determine actual VAT type from `LegalEntity.vat_reporting_frequency`.

5. `_validate_period_for_vat_type(period, vat_type)`:
   - `EXEMPT` → blocked.
   - `BIMONTHLY` + even month in period → blocked.

6. Conflict check: `work_item_repo.get_by_client_record_period(client_record_id, period)`.
   - Checks only non-deleted items (`deleted_at IS NULL`).
   - If exists → `ConflictError("VAT.CONFLICT")`.
   - **Warning**: if a FILED item is soft-deleted, a new item for the same period can be created (code comment documents this risk).

7. Determine initial status:
   - `mark_pending=True`: status = `PENDING_MATERIALS`. Requires non-empty `pending_materials_note`.
   - `mark_pending=False`: status = `MATERIAL_RECEIVED`.

8. `TaxCalendarMaterializationService.ensure_periodic_entry(ObligationType.VAT, period, months)`:
   - `months`: 1 for MONTHLY, 2 for BIMONTHLY (from `_VAT_PERIOD_MONTHS_COUNT`).
   - Find existing `TaxCalendarEntry` by (obligation_type, period, period_months_count).
   - If not found: compute `due_date` from deadline rule, insert in SAVEPOINT.
   - SAVEPOINT protects against concurrent inserts: on IntegrityError → rollback SAVEPOINT, refetch.

9. `work_item_repo.create(...)`:
   - Sets `tax_calendar_entry_id = tax_calendar_entry.id`.
   - Sets `due_date_original = tax_calendar_entry.due_date` (snapshot at creation time).
   - Sets `due_date_effective = tax_calendar_entry.due_date` (snapshot at creation time).

10. `materializer.link_vat_work_item(item)`:
    - Verifies `tax_calendar_entry_id` matches.
    - Backfills `due_date_original` and `due_date_effective` if still None.
    - `db.flush()`.

11. `db.flush()`.

12. `work_item_repo.append_audit(...)`:
    - Action: `ACTION_WORK_ITEM_CREATED_PENDING` or `ACTION_MATERIAL_RECEIVED`.
    - Writes audit row with status and period.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `vat` | Creates VatWorkItem, writes audit |
| `tax_calendar` | Find-or-creates TaxCalendarEntry |
| `clients` | Reads ClientRecord, LegalEntity (guard) |

## 5. Side Effects

| Entity | Effect |
|--------|--------|
| `VatWorkItem` | Created with initial status |
| `TaxCalendarEntry` | Find-or-created (shared global row per period + obligation type + frequency) |
| `VatWorkItem` audit | 1 row appended |
| `due_date_original` | Snapshot of TaxCalendarEntry.due_date at creation |
| `due_date_effective` | Snapshot of TaxCalendarEntry.due_date at creation |

## 6. Transaction Boundaries

All in one transaction.
`TaxCalendarEntry` creation uses SAVEPOINT for race safety.
`db.flush()` called twice (after `link_vat_work_item`, then again explicitly).
Commit is the caller's responsibility.

## 7. Idempotency / Duplicate Protection

Not idempotent: `ConflictError` on duplicate (same client, same period).
Soft-deleted FILED items are invisible to the conflict check — creating a replacement for a deleted FILED item is possible (documented risk in code).

`TaxCalendarEntry` creation is idempotent via SAVEPOINT + refetch.

## 8. Locks / Concurrency

No lock on `VatWorkItem` creation.
Concurrent creation of the same period for the same client is caught by DB unique constraint → IntegrityError → ConflictError.
`TaxCalendarEntry` SAVEPOINT protects against concurrent global entry creation.

## 9. Preconditions

| Condition | Check |
|-----------|-------|
| Client exists | Raises if not found |
| Client ACTIVE | Raises if CLOSED or FROZEN |
| VAT type not EXEMPT | Raises if EXEMPT |
| Period month matches frequency | BIMONTHLY requires odd month |
| No existing work item for (client, period) | Raises ConflictError |
| `pending_materials_note` if `mark_pending=True` | Raises if empty |
| DeadlineRule configured for period | Raises if no rule found |

## 10. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Client CLOSED | `VAT.CLIENT_CLOSED` | 409 |
| Client FROZEN | `VAT.CLIENT_FROZEN` | 409 |
| VAT type EXEMPT | `VAT.CLIENT_EXEMPT` | 409 |
| Invalid period for BIMONTHLY | `VAT.INVALID_PERIOD_FOR_FREQUENCY` | 422 |
| Existing work item for period | `VAT.CONFLICT` | 409 |
| `mark_pending=True` without note | `VAT.PENDING_NOTE_REQUIRED` | 422 |
| No DeadlineRule for period | `TAX_CALENDAR.DEADLINE_RULE_MISSING` | 500 |

## 11. Derived State

`due_date_original` and `due_date_effective` are **snapshots** of the TaxCalendarEntry at creation time.
Changes to `DeadlineRule` do not retroactively update existing work items.
`due_date_effective` can be manually overridden after creation (for extensions/extensions).

## 12. Tests

- `tests/vat/api/test_vat_reports_intake.py`
- `tests/vat/api/test_vat_reports_status.py`
- `tests/vat/api/test_vat_reports_materials_complete.py`
- `tests/clients/service/test_client_onboarding_orchestrator.py` (onboarding path)

## 13. Documentation Target

- `docs/domains/vat.md` — work item lifecycle, creation rules
- `docs/domains/tax-calendar.md` — TaxCalendarEntry materialization
