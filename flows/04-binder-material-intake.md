## Scope

This file owns only:
- Documentation of the binder material intake flow as it exists in the code.

Source of truth: reference

# Flow: Binder Material Intake

## 1. Trigger

`POST /binders/receive`

## 2. Entry Point in Code

```
backend/app/binders/api/binders_receive_return.py
  receive_binder()
    → BinderService.receive_binder()
      → BinderIntakeService.receive()
```

Main logic: `backend/app/binders/services/binder_intake_service.py`

## 3. Sequence

1. **Guards**:
   - `assert_client_record_is_active(client_record)` — client must not be CLOSED, FROZEN, or DELETED.
   - Check businesses: if any businesses exist and none are ACTIVE → `BINDER.CLIENT_LOCKED`.

2. **Resolve target binder** (`_resolve_existing_binder_for_materials`):
   - If active binder exists and has structured material periods:
     - Compute min/max period window from incoming materials.
     - If all incoming material is older than `active_binder.period_start`: search for an older IN_OFFICE binder whose stored period window contains the incoming window.
     - If found: use that older binder.
     - If not found: use the active binder (note required — see step 8).
   - Otherwise: use the active binder, or create a new one.

3. **Binder selection**:
   - If older/existing binder resolved: use it (`is_new_binder = False`).
   - Else if active binder exists and `open_new_binder=False`: use active binder (`is_new_binder = False`).
   - Else (new binder needed):
     - If `open_new_binder=True` and active binder exists: set `active_binder.period_end` (from last material's structured period), then `mark_full()`.
     - Require `office_client_number` set on client.
     - Create new `Binder` with `binder_number = f"{office_client_number}/{seq}"`.
     - Record creation as one `binder.created` `EntityAuditLog` row (Phase 5; was `BinderLifecycleLog` initial state).
     - `is_new_binder = True`.

4. **Lifecycle transition** (existing binders only):
   - `lifecycle_service.receive_material(binder, allow_full_in_office=True)` — advances binder location/capacity state.

5. **Old-period note validation** (`_validate_old_period_note`):
   - If any material's `period_year/period_month_start` is older than `binder.period_start`: require non-empty `notes` on the intake. Raises `BINDER.OLD_PERIOD_NOTE_REQUIRED`.

6. **Create intake record**:
   - Create `BinderIntake` (binder_id, received_at, received_by, notes).

7. **Create material rows**:
   - For each material dict: create `BinderIntakeMaterial` (material_type, business_id, annual_report_id, vat_report_id, period_year, period_month_start, period_month_end, description).

8. **Backfill `period_start`**:
   - If `binder.period_start is None` and materials exist: set from first material's `period_year/period_month_start`. `db.flush()`.

9. **VAT auto-advance** (`_auto_advance_vat_work_items`):
   - For each material where `material_type == "vat"` and `vat_report_id` is set:
     - `vat_repo.get_by_id_for_update(vat_id)` — row-level lock.
     - If `status == PENDING_MATERIALS`: update to `MATERIAL_RECEIVED`, write audit.
     - Skip duplicates (tracked via `seen` set), skip if status is not PENDING_MATERIALS.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `binders` | Creates BinderIntake, BinderIntakeMaterials; conditionally creates Binder; writes `binder.*` EntityAuditLog rows |
| `vat` | Conditionally updates VatWorkItem (PENDING_MATERIALS → MATERIAL_RECEIVED) |
| `clients` | Reads ClientRecord (guard) |
| `businesses` | Reads Business list (guard: all-locked check) |

## 5. Side Effects

| Entity | Effect |
|--------|--------|
| `BinderIntake` | Always created |
| `BinderIntakeMaterial` | Created for each material in the request |
| `Binder` | Created if new binder needed |
| `EntityAuditLog` (`binder.*`) | `binder.created` on new binder; `binder.material_received` / lifecycle actions on transitions (Phase 5; was `BinderLifecycleLog`) |
| `Binder.period_start` | Backfilled from first material if was None |
| `Binder.period_end` | Set on old active binder when `open_new_binder=True` |
| `VatWorkItem.status` | Changed PENDING_MATERIALS → MATERIAL_RECEIVED for linked VAT items |
| VAT audit | Appended for each advanced VAT work item |

## 6. Transaction Boundaries

All in one transaction (caller session).
`db.flush()` after `period_start` backfill.
Each VAT work item advance uses `get_by_id_for_update()` (row-level lock per item, released at transaction commit).

## 7. Idempotency / Duplicate Protection

Not idempotent: each call creates a new `BinderIntake` row.

VAT auto-advance is safe against duplicates:
- `seen` set prevents processing the same `vat_report_id` twice in one intake.
- Status check (`== PENDING_MATERIALS`) prevents double-advancing.

`period_start` backfill is safe: only sets if `None`.

## 8. Locks / Concurrency

`get_by_id_for_update()` per VAT work item during auto-advance — row-level lock per item.
No lock on the `Binder` row itself. Concurrent intakes for the same client could create duplicate binders (race condition on `open_new_binder=True`).

## 9. Preconditions

- Client must be ACTIVE.
- At least one ACTIVE business must exist (if any businesses exist).
- If creating a new binder: `office_client_number` must be set on the client.
- If material is for an older period than the binder's `period_start`: `notes` must be non-empty.

## 10. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Client not active | `CLIENT_RECORD.NOT_ACTIVE` | 409 |
| All businesses locked | `BINDER.CLIENT_LOCKED` | 409 |
| Missing `office_client_number` | `BINDER.OFFICE_NUMBER_MISSING` | 409 |
| Old-period material without note | `BINDER.OLD_PERIOD_NOTE_REQUIRED` | 422 |

## 11. Derived State

None. All writes are to DB rows.

## 12. Tests

- `tests/binders/service/test_binder_intake_service_additional.py`
- `tests/binders/api/test_binder_intakes_additional.py`
- `tests/binders/api/test_binders.py`
- `tests/regression/test_core_regressions_binders_charges_notifications.py`

**GAP**: No dedicated test for the VAT auto-advance path (`material_type == "vat"` + `vat_report_id` linked to a PENDING_MATERIALS item).

## 13. Intake Edit Flow

`PATCH /api/v1/binders/{binder_id}/intakes/{intake_id}`

Entry point: `backend/app/binders/api/binders_audit.py:patch_binder_intake`
Service: `backend/app/binders/services/binder_intake_edit_service.py:BinderIntakeEditService.edit_intake`

Patchable fields (all optional):
- `received_at`, `received_by`, `notes` — field-level update with audit log
- `client_record_id`, `binder_id` — transfer to another binder/client
- `business_ids`, `annual_report_ids`, `vat_report_ids` — FK replacement with cross-client ownership validation

The `binder_id` path param scopes the lookup; a mismatched intake returns 404. All changes write one `binder_intake.updated` `EntityAuditLog` row per mutated field (Phase 5; was `BinderIntakeEditLog`).

## 14. Documentation Target

- `docs/domains/binders.md` — intake flow, period resolution, VAT auto-advance, intake edit
- `docs/domains/vat.md` — PENDING_MATERIALS → MATERIAL_RECEIVED via binder intake
