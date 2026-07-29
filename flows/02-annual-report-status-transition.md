## Scope

This file owns only:
- Documentation of the annual report status transition flow as it exists in the code.

Source of truth: reference

# Flow: Annual Report Status Transition

## 1. Trigger

`POST /annual-reports/{report_id}/status` (advisor only).

`POST /annual-reports/{report_id}/transition` and the `ReportStage` layer **retired in W2** (D-40) —
a lossy alias over statuses that no longer had anything to abstract once all three domains shared one
six-value enum.

## 2. Entry Point in Code

```
backend/app/annual_reports/services/annual_report_status_service.py
  AnnualReportStatusService.transition_status()
    → repo.get_by_id_for_update()      # row-level lock
    → assert_transition_allowed()      # the shared graph owns the rule
    → _assert_filing_readiness()       # only on → SUBMITTED
    → repo.update()                    # status + closing facts
    → EntityAuditWriter.record_status_change()
```

There are no cross-domain side effects. The signature flow that used to hang off stage 4 retired
with D-5 (W4-pre).

## 3. Valid Transitions

Owned by the shared graph in `backend/app/common/obligation_lifecycle.py`, not by this domain.

The ladder, in order:

```
awaiting_input → input_received → in_progress → awaiting_verification → submitted
```

`canceled` sits **off** the ladder — reachable from any unlocked stage, and terminal.

Rules the graph enforces:

- **One step at a time.** Skipping a stage raises; a caller that must advance several stages on one
  event calls the graph once per step so each transition is validated and audited on its own.
- **Backward moves require a reason.** Forward moves do not.
- **`submitted` is locked** — no outgoing transitions at all. Correcting a submitted report goes
  through an amendment (W4), not a transition.
- **`canceled` is terminal.**

For an annual report, `awaiting_verification` means *ready, awaiting internal review* — the same
meaning it carries in VAT. No client sees the report; the office reviews and files alone (D-5).

**Stage 2 (`input_received`) has no gate for annual reports yet.** The intended rule — the tax year's
VAT periods all closed **and** the year's documents marked received (D-18) — lands in W7. Until then
the stage is stepped through manually.

## 4. Sequence

1. `repo.get_by_id_for_update(report_id)` — row-level lock (`SELECT FOR UPDATE`).
2. Validate `new_status` is a known `ObligationStatus` value.
3. `assert_transition_allowed(current, target, reason=note)` — the shared graph (see §3).
4. **Only on `→ SUBMITTED`**: `_assert_filing_readiness()` delegates to
   `AnnualReportReadinessService.get_readiness_check()` and raises listing every blocking issue.
5. Build the update dict. **On `→ SUBMITTED`** this is the closing act and writes all of it:
   - `closed_at` (the supplied time, else now) and `closed_by` (the acting user — never NULL, since
     only an advisor can perform this transition).
   - Optionally `ita_reference` and `submission_method`; if the deadline type is `STANDARD` and a
     submission method is given, `filing_deadline` is recalculated.
   - `closed_late` — the **Israel-local** date of `closed_at` against the deadline *as recalculated
     by this very submission*, NULL when there is no deadline (D-20, D-32).
   - `assessment_amount`, `refund_due`, `tax_due` when provided. These used to require a separate
     `closed` status after `submitted`; the two were one act, so they merged.
6. `repo.update(report_id, report=report, **update_fields)`.
7. `EntityAuditWriter.record_status_change(ENTITY_ANNUAL_REPORT, ...)` — an `EntityAuditLog` row with
   old/new status, note, `client_record_id`, `tax_year`, actor id, actor display-name snapshot, and
   on a close also `closed_by` and `closed_late`.

## 5. Domains Involved

| Domain | Role |
|--------|------|
| `annual_reports` | Updates AnnualReport |
| `common` | Owns the transition graph and the closing-fact helpers |
| `audit` | Writes generic EntityAuditLog |

## 6. Side Effects

- Updates the `AnnualReport` row: status, and on a close the full set of closing facts (§4 step 5).
- Creates one `EntityAuditLog` row per successful transition.

No signature requests, no notifications, no cross-domain writes.

## 7. Transaction Boundaries

All operations run in one DB transaction (caller session via `get_db()`). The row-level lock is held
until commit. **No savepoints** — the savepoint that used to isolate this transition existed only
because a client signature in an outer transaction had to survive a failed report transition.

## 8. Idempotency / Duplicate Protection

Not idempotent. Each successful transition appends a new `EntityAuditLog` row; an invalid or failed
audit write rolls back the status mutation.

## 9. Locks / Concurrency

`repo.get_by_id_for_update()` issues `SELECT FOR UPDATE` on the `annual_reports` row, preventing
concurrent transitions on the same report. Audit rows are inserted without an explicit lock.

## 10. Preconditions

- Report must exist.
- Transition must satisfy the shared graph (§3).
- `→ SUBMITTED`: all four readiness gates must pass — assignee set, required schedules complete,
  total income greater than zero, and either `tax_due` or `refund_due` persisted.

## 11. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Report not found | `ANNUAL_REPORT.NOT_FOUND` | 404 |
| Unknown status value | `ANNUAL_REPORT.INVALID_STATUS` | 409 |
| Same-status, skipped stage, or move out of `canceled` | `OBLIGATION.INVALID_TRANSITION` | 409 |
| Any transition out of `submitted` | `OBLIGATION.LOCKED` | 409 |
| Backward move without a reason | `OBLIGATION.TRANSITION_REASON_REQUIRED` | 400 |
| `→ SUBMITTED` when not ready | `ANNUAL_REPORT.INVALID_STATUS` | 409 |

## 12. Derived State

None. All state changes are written to the DB immediately. `closed_late` is recorded **at the close**
and is not recomputed afterwards (D-20).

## 13. Tests

- `tests/annual_reports/service/test_status_service_additional.py`
- `tests/annual_reports/api/test_annual_report_closing_and_lock.py`
- `tests/annual_reports/service/test_annual_report_delete_report.py`
- `tests/notifications/service/test_notification_policy_annual_report.py`

## 14. Documentation Target

- `docs/domains/annual-reports.md` — status machine, filing readiness, closing and locking
- `docs/tax-lifecycle-refactor-progress.md` — the wave record for the shared lifecycle
