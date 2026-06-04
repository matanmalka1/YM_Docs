## Scope

This file owns only:
- Documentation of the annual report status transition flow as it exists in the code.

Source of truth: reference

# Flow: Annual Report Status Transition

## 1. Trigger

`PATCH /annual-reports/{report_id}/status` or `PATCH /annual-reports/{report_id}/stage`.

Stage transitions map to status via `STAGE_TO_STATUS` (see §3).

## 2. Entry Point in Code

```
backend/app/annual_reports/services/status_service.py
  AnnualReportStatusService.transition_status()
    → repo.get_by_id_for_update()      # row-level lock
    → _assert_filing_readiness()       # only on → SUBMITTED
    → repo.update()
    → repo.append_status_history()
    → EntityAuditWriter.record_status_change()
    → _cancel_pending_signature_requests()   # conditional
    → _trigger_signature_request()           # conditional
```

Stage shortcut: `transition_stage()` → maps stage to status → calls `transition_status()`.

## 3. Valid Transitions

Defined in `backend/app/annual_reports/services/constants.py`.

```
NOT_STARTED     → COLLECTING_DOCS
COLLECTING_DOCS → IN_PREPARATION | NOT_STARTED
IN_PREPARATION  → PENDING_CLIENT | COLLECTING_DOCS
PENDING_CLIENT  → IN_PREPARATION | SUBMITTED
SUBMITTED       → IN_PREPARATION | CLOSED
CLOSED          → (terminal, no transitions)
CANCELED        → (terminal, no transitions)
```

Stage shortcut map:
```
material_collection → collecting_docs
in_progress         → in_preparation
final_review        → in_preparation
client_signature    → pending_client
transmitted         → submitted
```

## 4. Sequence

1. `repo.get_by_id_for_update(report_id)` — row-level lock (`SELECT FOR UPDATE`).
2. Validate `new_status` is a known enum value.
3. Validate transition is allowed via `VALID_TRANSITIONS[current_status]`.
4. **Only on `→ SUBMITTED`**: run `_assert_filing_readiness()`.
   - Delegates to `AnnualReportReadinessService.get_readiness_check()`.
   - Raises `AppError` listing all blocking issues if not ready.
5. Build update fields dict:
   - `→ SUBMITTED`: set `submitted_at` (now if not provided), optionally `ita_reference`, `submission_method`. If deadline type is STANDARD and `submission_method` given: recalculate `filing_deadline`.
   - `→ CLOSED`: set `assessment_amount`, `refund_due`, `tax_due` if provided.
6. `repo.update(report_id, report=report, **update_fields)` — update row.
7. `repo.append_status_history(...)` — insert `AnnualReportStatusHistory` row (always).
8. `EntityAuditWriter.record_status_change(ENTITY_ANNUAL_REPORT, ...)` — insert audit row (always).
9. **If leaving `PENDING_CLIENT`** (old == PENDING_CLIENT, new != PENDING_CLIENT):
   - `_cancel_pending_signature_requests()` — cancel all pending SRs linked to this report.
10. **If entering `PENDING_CLIENT`** (new == PENDING_CLIENT):
    - `_cancel_pending_signature_requests()` — cancel any existing pending SRs first (re-entry case).
    - `_trigger_signature_request()`:
      - Read `ClientRecord` via `ClientRecordRepository.get_by_id()` — raises `CLIENT_RECORD.NOT_FOUND` if missing (blocks transition before DB write).
      - Resolve `signer_name` via `ClientRecordRepository.get_signer_name_by_legal_entity_id()`: returns `Person.full_name` (OWNER link) or `LegalEntity.official_name`. Raises `ANNUAL_REPORT.SIGNER_NAME_MISSING` if neither exists.
      - Create `SignatureRequest` (type: `ANNUAL_REPORT_APPROVAL`, `business_id=None`, 14-day expiry, linked to report).

## 5. Domains Involved

| Domain | Role |
|--------|------|
| `annual_reports` | Updates AnnualReport, creates AnnualReportStatusHistory |
| `signature_requests` | Creates or cancels SignatureRequest |
| `clients` | Reads ClientRecord and resolves signer name from LegalEntity/Person |
| `audit` | Writes AuditLog |

## 6. Side Effects

- Updates: `AnnualReport` row (status, optionally submitted_at, ita_reference, submission_method, filing_deadline, assessment_amount, refund_due, tax_due).
- Creates: `AnnualReportStatusHistory` row on every call.
- Creates: `AuditLog` row on every call.
- **On `PENDING_CLIENT` entry**: creates 1 `SignatureRequest`.
- **On `PENDING_CLIENT` exit** (to any state): cancels all pending `SignatureRequest` rows linked to this report.
- **On re-entry to `PENDING_CLIENT`**: cancels old SRs, then creates a new one.

## 7. Transaction Boundaries

All operations run in one DB transaction (caller session via `get_db()`).
The row-level lock (`SELECT FOR UPDATE`) is held until the transaction commits.
No savepoints used in this flow.

## 8. Idempotency / Duplicate Protection

Not idempotent.
Each call appends a new `AnnualReportStatusHistory` row and a new `AuditLog` row, regardless of whether the status is actually changing.

Re-entering `PENDING_CLIENT` is safe: old pending SRs are cancelled before a new SR is created.

## 9. Locks / Concurrency

`repo.get_by_id_for_update()` issues `SELECT FOR UPDATE` on the `annual_reports` row.
Prevents concurrent transitions on the same report.
All other accessed rows (audit, history, SR) are inserted without explicit lock.

## 10. Preconditions

- Report must exist.
- Transition must be in `VALID_TRANSITIONS[current_status]`.
- `→ SUBMITTED`: all readiness checks must pass (delegated to `AnnualReportReadinessService`).
- `→ PENDING_CLIENT`: client record must exist and must have a resolvable signer name (Person.full_name or LegalEntity.official_name).

## 11. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Report not found | `ANNUAL_REPORT.NOT_FOUND` | 404 |
| Invalid status value | `ANNUAL_REPORT.INVALID_STATUS` | 409 |
| Transition not in VALID_TRANSITIONS | `ANNUAL_REPORT.INVALID_STATUS` | 409 |
| `→ SUBMITTED` when not ready | `ANNUAL_REPORT.INVALID_STATUS` | 409 |
| `→ PENDING_CLIENT` — client record missing | `CLIENT_RECORD.NOT_FOUND` | 404 |
| `→ PENDING_CLIENT` — no resolvable signer name | `ANNUAL_REPORT.SIGNER_NAME_MISSING` | 400 |
| Invalid stage name | `ANNUAL_REPORT.INVALID_STAGE` | 409 |
| Amend on non-SUBMITTED report | `ANNUAL_REPORT.INVALID_STATUS_FOR_AMEND` | 409 |

## 12. Derived State

None. All state changes are written to the DB immediately.

## 13. Tests

- `tests/annual_reports/api/test_annual_report_stage_transition.py`
- `tests/annual_reports/service/test_status_service_additional.py`
- `tests/annual_reports/service/test_annual_report_delete_report.py`
- `tests/signature_requests/` (covers SR creation/cancellation on status transition)
- `tests/notification/service/test_notification_policy_annual_report.py`

Tests added covering: `pending_client` with no business succeeds, signer name resolved from Person, missing client record blocks before DB write.

## 14. Documentation Target

- `docs/domains/annual-reports.md` — status machine, filing readiness, SR lifecycle
- `docs/domains/signature-requests.md` — SR creation and cancellation triggers
