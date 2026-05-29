## Scope

This file owns only:
- Documentation of the client freeze/close cascade as it exists in the code.

Source of truth: reference

# Flow: Client Freeze / Close Cascade

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
      → BinderRepository.close_in_office_by_client_record()
```

## 3. Sequence

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
        - Bulk-update all IN_OFFICE binders → `capacity_status = FULL`. No per-binder history written.
5. If obligation-trigger fields changed: `generate_client_obligations(..., best_effort=True)`.
6. `EntityAuditWriter.record_update(ENTITY_CLIENT, ...)` — write client audit.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `clients` | Updates ClientRecord.status |
| `vat_reports` | Bulk-cancels VatWorkItems |
| `annual_reports` | Bulk-cancels AnnualReports |
| `binders` | Bulk-marks IN_OFFICE binders as FULL |
| `audit` | Writes client audit |

## 5. Side Effects

| Entity | Effect |
|--------|--------|
| `ClientRecord.status` | Set to FROZEN or CLOSED |
| All non-FILED `VatWorkItem` for client | Status → CANCELED (bulk, no per-item audit) |
| All non-terminal `AnnualReport` for client | Status → CANCELED (bulk, no per-report history) |
| All IN_OFFICE `Binder` rows | `capacity_status` → FULL (no new intake; location stays IN_OFFICE) |
| `AuditLog` | 1 client-level update entry |

**Not written on cascade**:
- No `AnnualReportStatusHistory` rows for bulk-canceled reports.
- No per-item `VatWorkItem` audit rows.
- No per-binder history or audit rows.
- No `SignatureRequest` cancellation for linked pending SRs.

## 6. Transaction Boundaries

All in one transaction. Bulk cancellations use `db.flush()` but no savepoints.

## 7. Idempotency

- VAT bulk-cancel: safe — filters `status NOT IN [FILED]`; FILED items are never touched.
- Annual report bulk-cancel: safe — filters `status NOT IN [SUBMITTED, CLOSED, CANCELED]`.
- Binder close: safe — sets `capacity_status = FULL` regardless of current value; idempotent.

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

## 11. Derived State

Work queue items for the frozen/closed client are bulk-cancelled as part of the cascade.
The active-client scope filter (`scope_to_active_clients_stmt`) excludes CLOSED/FROZEN clients from aggregate work queue views.

## 12. Tests

- `tests/clients/service/test_client_service_mutations.py`
- `tests/clients/api/test_clients_mutations_additional.py`
- `tests/clients/service/test_client_freeze_close_cascade.py` — full cascade integration tests

## 13. Documentation Target

- `docs/domains/clients.md` — freeze/close behavior and cascade
- `docs/domains/vat-reports.md` — bulk cancel on client close
- `docs/domains/annual-reports.md` — bulk cancel on client close
- `docs/domains/binders.md` — close-in-office on client close
