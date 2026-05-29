## Scope

This file owns only:
- Documentation of the client + business creation cascade flow as it exists in the code.

Source of truth: reference

# Flow: Client + Business Creation Cascade

## 1. Trigger

`POST /clients/` API endpoint.
Called by `CreateClientService.create_from_request()`.

## 2. Entry Point in Code

```
backend/app/clients/services/create_client_service.py
  CreateClientService.create_from_request()
    → CreateClientService.create_client()
      → CreateClientService._create_client_identity()
        → ClientOnboardingOrchestrator.run()        # binder + obligations
      → BusinessService.create_business_for_client_record()
```

## 3. Sequence

### Phase 1 — Identity creation (`_create_client_identity`)

1. Validate no active or deleted client exists for the same `id_number`.
2. Validate `entity_type != EMPLOYEE`, `COMPANY_LTD` requires `CORPORATION` id type.
3. Normalize `vat_reporting_frequency` and `vat_exempt_ceiling` based on entity type.
4. Create `LegalEntity`.
5. Create `Person` linked to the `LegalEntity`.
6. Create `ClientRecord` linked to the `LegalEntity`.
7. Call `ClientOnboardingOrchestrator.run()` (see Phase 2).
8. Write audit: `record_create(ENTITY_CLIENT)`.

### Phase 2 — Onboarding orchestration (`ClientOnboardingOrchestrator.run`)

File: `backend/app/clients/services/client_onboarding_orchestrator.py`

1. **Initial binder** — `_ensure_initial_binder()`:
   - If client already has any binder: skip.
   - Else: `create_initial_binder()` → create `Binder` + `BinderLifecycleLog`.
   - Requires `actor_id` and `office_client_number` (both raise `ValueError` if None).

2. **Annual reports** — `generate_client_obligations()`:
   - Compute target years: `_years_to_generate()` → current year; if current month ≥ `CLIENT_OBLIGATION_NEXT_YEAR_START_MONTH`, also next year.
   - For each year — in SAVEPOINT:
     - `AnnualReportService.create_report()`.
     - On `ConflictError` (report exists): rollback SAVEPOINT, skip.
     - On other error: rollback SAVEPOINT, raise (hard failure in create flow).

3. **VAT work items** — `_sync_vat_work_items()`:
   - Compute VAT obligation plan from `LegalEntity.vat_reporting_frequency`.
   - For each period in each year:
     - `TaxCalendarMaterializationService.ensure_periodic_entry()` — find-or-create `TaxCalendarEntry`.
     - If `entry.due_date < reference_date`: skip (past due).
     - If no existing VatWorkItem for (client, period): `create_work_item(..., mark_pending=True)`.
     - On `ConflictError`: skip.
   - If `vat_reporting_frequency` is None: no VAT work items created.

4. **Advance payments** — `_sync_advance_payments()`:
   - If `advance_payment_frequency` is None: skip entirely.
   - For each period in each year:
     - `ensure_periodic_entry()` — find-or-create `TaxCalendarEntry`.
     - If past due: skip.
     - If no existing `AdvancePayment`: `create_payment_for_client()`.
     - If existing with wrong `period_months_count` or `due_date`: update in place.

### Phase 3 — Business creation (`create_business_for_client_record`)

File: `backend/app/businesses/services/business_service.py`

1. Guard: if all existing non-deleted businesses are CLOSED → reject (`BUSINESS.CLIENT_ALL_CLOSED`).
2. Guard: duplicate `business_name` (case-insensitive) → reject (`BUSINESS.NAME_CONFLICT`).
3. Create `Business`.
4. Write audit: `record_create(ENTITY_BUSINESS)`.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `clients` | Creates LegalEntity, Person, ClientRecord |
| `businesses` | Creates Business |
| `binders` | Creates Binder, BinderLifecycleLog |
| `annual_reports` | Creates AnnualReport (1–2 rows) |
| `vat_reports` | Creates VatWorkItem rows |
| `advance_payments` | Creates AdvancePayment rows |
| `tax_calendar` | Find-or-creates TaxCalendarEntry (shared global rows) |
| `audit` | Writes AuditLog (CLIENT, BUSINESS) |

## 5. Side Effects

- Creates: `LegalEntity`, `Person`, `ClientRecord`, `Business`
- Creates: `Binder` + `BinderLifecycleLog` (if no binder existed)
- Creates: 1–2 `AnnualReport` rows (current year + optionally next year)
- Creates: N `VatWorkItem` rows (based on VAT frequency and obligation plan)
- Creates: N `AdvancePayment` rows (based on advance payment frequency)
- Find-or-creates: `TaxCalendarEntry` rows (shared global; reused across clients)
- Writes: 2 audit entries (ENTITY_CLIENT, ENTITY_BUSINESS)

## 6. Transaction Boundaries

All operations run in a single DB transaction owned by `get_db()`.

- Commit on success → all rows persisted.
- Any unhandled exception → full rollback.
- Each annual report year uses an independent SAVEPOINT.
  - ConflictError on a year → SAVEPOINT rolled back, other years unaffected.
  - Non-conflict error in `create_flow` mode → SAVEPOINT rolled back, error re-raised → outer transaction rolls back.

## 7. Idempotency / Duplicate Protection

| Step | Protection |
|------|------------|
| Annual reports | ConflictError caught → SAVEPOINT rollback → skipped |
| VAT work items | `get_by_client_record_period()` check before create; ConflictError swallowed |
| Advance payments | Checked before create; if exists with wrong period/due_date → updated |
| Initial binder | `count_all_by_client() > 0` guard → skipped if binder exists |
| `LegalEntity` / `ClientRecord` | IntegrityError → ConflictError |

## 8. Locks / Concurrency

- No row-level lock on client creation.
- Concurrent creates with same `id_number`: DB unique constraint → IntegrityError → ConflictError (409).
- Concurrent duplicate VAT work item creation: no lock; ConflictError caught at application level.
- `TaxCalendarEntry` creation: SAVEPOINT-protected against concurrent inserts (IntegrityError → rollback SAVEPOINT → refetch).

## 9. Preconditions

- `entity_type != EMPLOYEE`.
- `id_number` not already active or soft-deleted.
- If `entity_type == COMPANY_LTD`: `id_number_type` must be `CORPORATION`.
- `actor_id` must not be None (required for binder creation).
- `office_client_number` must be set on `ClientRecord` after creation (required for binder number). If missing, `ValueError` is raised.

## 10. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| `entity_type == EMPLOYEE` | — | 400 (ValueError) |
| Duplicate `id_number` (active) | `CLIENT.CONFLICT` | 409 |
| Duplicate `id_number` (deleted) | `CLIENT.DELETED_EXISTS` | 409 |
| Missing `actor_id` for binder | — | 500 (ValueError) |
| Missing `office_client_number` | — | 500 (ValueError) |
| All businesses CLOSED for client | `BUSINESS.CLIENT_ALL_CLOSED` | 409 |
| Duplicate `business_name` | `BUSINESS.NAME_CONFLICT` | 409 |

## 11. Derived State

None at creation. Work queue items referencing newly created VAT work items, annual reports, and advance payments are derived at query time.

## 12. Tests

- `tests/clients/service/test_client_onboarding_orchestrator.py`
- `tests/clients/api/test_onboarding_layer1.py`
- `tests/clients/service/test_client_creation_service.py`
- `tests/clients/service/test_advance_payment_frequency_onboarding.py`
- `tests/clients/api/test_clients.py`

**GAP**: No test covers the partial-success behavior of annual report SAVEPOINT failure mid-flow (e.g., year 1 succeeds, year 2 fails).

## 13. Documentation Target

- `docs/domains/clients.md` — client creation behavior
- `docs/domains/annual-reports.md` — obligation generation
- `docs/domains/binders.md` — initial binder creation
- `docs/domains/vat-reports.md` — onboarding VAT work items
