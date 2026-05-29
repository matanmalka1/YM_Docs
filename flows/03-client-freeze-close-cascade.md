## Scope

This file owns only:
- Documentation of the client freeze/close cascade as it exists in the code.

Source of truth: reference

# Flow: Client Freeze / Close Cascade

## ⚠️ CRITICAL BUG

`BinderRepository.close_in_office_by_client_record()` is called at:
```
backend/app/clients/services/client_update_service.py:84
```
This method **does not exist** in `BinderRepository`.

**Runtime effect**: When any client is frozen or closed, the call raises `AttributeError`.
The entire DB transaction rolls back. The client status is NOT updated. Nothing is cancelled.

**The cascade is completely broken until this method is implemented.**

## 1. Trigger

`PATCH /clients/{client_id}` with `{ "status": "frozen" }` or `{ "status": "closed" }`.

## 2. Entry Point in Code

```
backend/app/clients/services/client_update_service.py
  ClientUpdateService.update_client()
    → _update_client_record_status(client_id, new_status)
      → record_repo.update_status(record.id, new_status)
      → VatWorkItemClientStatusService.cancel_open_by_client_record()
      → AnnualReportClientStatusService.cancel_open_by_client_record()
      → BinderRepository.close_in_office_by_client_record()  ← BUG: method missing
```

## 3. Intended Sequence (as written, ignoring the bug)

1. `get_full_record(db, client_id)` — read full client snapshot.
2. If `entity_type` is changing:
   - Require `ADVISOR` role.
   - Write audit for entity type change.
3. `apply_graph_update(db, client_id, **fields)` — update `LegalEntity`, `Person`, `ClientRecord` fields.
4. If `status` in fields: `_update_client_record_status(client_id, new_status)`:
   a. `record_repo.update_status(record.id, new_status)` — update `ClientRecord.status`.
   b. If `new_status` is `CLOSED` or `FROZEN`:
      - `VatWorkItemClientStatusService.cancel_open_by_client_record(record.id)`:
        - Bulk-update all non-FILED VatWorkItems → `CANCELED`. No per-item history or audit written.
      - `AnnualReportClientStatusService.cancel_open_by_client_record(record.id)`:
        - Bulk-update all non-terminal AnnualReports → `CANCELED`. No per-report status history or audit written.
      - `BinderRepository(self.db).close_in_office_by_client_record(record.id)`:
        - **Method does not exist → AttributeError**.
5. If obligation-trigger fields changed: `generate_client_obligations(..., best_effort=True)`.
6. `EntityAuditWriter.record_update(ENTITY_CLIENT, ...)` — write client audit.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `clients` | Updates ClientRecord.status |
| `vat_reports` | Bulk-cancels VatWorkItems |
| `annual_reports` | Bulk-cancels AnnualReports |
| `binders` | Intended: close IN_OFFICE binders (BROKEN) |
| `audit` | Writes client audit |

## 5. Side Effects (when working correctly — currently blocked by bug)

| Entity | Effect |
|--------|--------|
| `ClientRecord.status` | Set to FROZEN or CLOSED |
| All non-FILED `VatWorkItem` for client | Status → CANCELED (bulk, no per-item audit) |
| All non-terminal `AnnualReport` for client | Status → CANCELED (bulk, no per-report history) |
| IN_OFFICE `Binder` rows | Intended to be closed (method missing) |
| `AuditLog` | 1 client-level update entry |

**Not written on cascade**:
- No `AnnualReportStatusHistory` rows for bulk-canceled reports.
- No per-item `VatWorkItem` audit rows.
- No `SignatureRequest` cancellation for linked pending SRs.

## 6. Transaction Boundaries

All in one transaction. Bulk cancellations use `db.flush()` but no savepoints.
Due to the bug, the entire transaction rolls back before any state is persisted.

## 7. Idempotency

- VAT bulk-cancel: safe — filters `status NOT IN [FILED]`; FILED items are never touched.
- Annual report bulk-cancel: safe — filters `status NOT IN [SUBMITTED, CLOSED, CANCELED]`.
- Due to the bug, the whole operation fails before any of this runs.

## 8. Locks / Concurrency

No row-level lock on `ClientRecord` during the update.
Bulk cancellations are not locked — concurrent reads could see inconsistent state mid-update.
Race condition risk if two concurrent requests attempt to freeze the same client.

## 9. Preconditions

- Client must exist.
- Entity type change requires `UserRole.ADVISOR`.

## 10. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Client not found | `CLIENT.NOT_FOUND` | 404 |
| Entity type change by SECRETARY | `CLIENT.ENTITY_TYPE_CHANGE_FORBIDDEN` | 403 |
| **Always**: missing binder method | — | 500 (AttributeError) |

## 11. Derived State

Work queue items for the frozen/closed client continue to appear until they are cancelled. Due to the bug, cancellation never executes, so work queue items from frozen/closed clients may remain visible.

The active-client scope filter (`scope_to_active_clients_stmt`) is used in several work queue queries and should exclude CLOSED/FROZEN clients from aggregate views — but this only works if the status is actually updated, which the bug prevents.

## 12. Tests

- `tests/clients/service/test_client_service_mutations.py`
- `tests/clients/api/test_clients_mutations_additional.py`

**GAP**: No test that exercises the full cascade (freeze → verify VatWorkItems canceled + AnnualReports canceled + binder closed). The bug would be caught by such a test.

## 13. Documentation Target

- `docs/domains/clients.md` — freeze/close behavior and cascade
- `docs/domains/vat-reports.md` — bulk cancel on client close
- `docs/domains/annual-reports.md` — bulk cancel on client close
- `docs/domains/binders.md` — close-in-office on client close (once fixed)
- `docs/project/security-findings.md` — bug tracking
