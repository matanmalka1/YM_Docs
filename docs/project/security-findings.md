## Scope
This file owns only:
- A central index of code findings (security + correctness) surfaced during domain-docs authoring.

This file must not contain:
- The fixes themselves (these are code changes, tracked separately).
- Domain documentation (lives in `docs/domains/*`).

Source of truth: reference

# Security & Code Findings

Central tracker for code issues found while writing canonical domain docs. Each domain doc has its own `## Known issues` section; this file aggregates them for prioritization. These are **code** problems — docs-only authoring does not fix them.

Status legend: `open` / `in-progress` / `fixed`.

## Findings

| ID | Severity | Domain | Issue | Location | Status |
|----|----------|--------|-------|----------|--------|
| F-001 | High (IDOR) | annual-reports | Income/expense line update & delete check only that the report exists, not that the line belongs to it. A valid `line_id` from another report can be mutated/deleted via an unrelated `report_id`. | `app/annual_reports/services/financial_service.py:172-183, 268-283` | open |
| F-002 | Medium | annual-reports | Transition to `pending_client` silently skips signature creation when client/business not found, while the status change still succeeds. | `app/annual_reports/services/status_signature_helper.py:42` | open |
| F-003 | Low | annual-reports | Legacy docs expect tax-calendar/reminder sync on status transitions; `transition_status` does status/history/audit/signatures only — no sync. | `app/annual_reports/services/status_service.py:82` | open |
| F-004 | Low / design | annual-reports | VAT auto-populate aggregates by `client_record_id`+`tax_year`, not per business — risky for multi-business clients if business-specific import is expected. | `app/annual_reports/services/vat_import_service.py:119` | open |
| F-005 | Medium | advance-payments | `timing_status` / `paid_late` compute overdue/late from legacy `due_date`, not `due_date_effective` — contradicts INV-05. Overridden effective dates produce wrong signals. | `app/advance_payments/schemas/advance_payment.py:49-51, 57-60` | open |
| F-006 | Medium | businesses | `constants.py` imports `EntityType` from `models/business.py`, which does not define it — `ImportError` whenever the module loads. Should import from the legal entities model. | `app/businesses/constants.py:1` | open |
| F-007 | Medium | vat-reports | Filing does not enforce `assigned_to` before moving to `filed`, contradicting INV-09. | `app/vat_reports/services/filing.py:58-99` | open |
| F-008 | High (IDOR) | vat-reports | Invoice update passes `business_activity_id` through without the ownership check that create enforces — allows attaching an activity from another legal entity. | `app/vat_reports/services/data_entry_invoice_update.py:81-121` (cf. create `data_entry_invoices.py:88-94`) | open |
| F-009 | Low | vat-reports | Several VAT service errors bypass the `DOMAIN.REASON` format (`AMENDED_ITEM_NOT_FOUND`, `MISSING_FINAL_AMOUNT`, `INVALID_NET_AMOUNT`, etc.). | `app/vat_reports/services/filing.py:31-40, 72-73`, `data_entry_invoice_update.py:76-77` | open |
| F-010 | Low | vat-reports | `is_amendment=True` accepted without `amends_item_id` — amendment validation only runs when the link is present. | `app/vat_reports/services/filing.py:55-65` | open |
| F-011 | Low / gap | binders | `BinderIntakeEditService.edit_intake` is fully implemented with cross-client FK ownership validation but no HTTP endpoint exists in openapi.json — capability inaccessible via API. | `app/binders/services/binder_intake_edit_service.py:57` | open |
| F-012 | Low | tasks | `GET /api/v1/tasks` declares `assigned_role` and `source_domain` as plain `str` instead of `UserRole`/validated source-domain enum — invalid values bypass validation and silently return empty results. | `app/tasks/api/routes.py:31-32`, `app/tasks/services/task_service.py:50-52` | open |
| F-013 | Low | charge | `ChargeListResponse.stats` is computed on a broader slice than the applied list filters — UI counters disagree with the visible rows. | `app/charge/services/charge_query_service.py` | open |
| F-014 | Low | charge | `POST /api/v1/charges/bulk-action` openapi declares `X-Idempotency-Key` as `required=false`, but runtime `require_idempotency_key` dependency returns 400 when missing. Spec/runtime mismatch. | `app/charge/api/charge.py:134` (openapi.json bulk-action params) | open |
| F-015 | High (broken feature) | reminders | `ReminderExecutorService._execute` unconditionally sets status to `FAILED` with an "not implemented" reason — no action type is actually executed. All reminders are no-ops. | `app/reminders/services/reminder_executor_service.py:46-60` | open |
| F-016 | High (broken feature) | reminders | No background job calls `ReminderExecutorService.fire_due()` — even if execution were implemented, nothing would trigger it. Scheduled reminders sit forever. | `app/core/background_jobs.py`, `app/lifespan.py` (absence) | open |
| F-017 | Low | notes | `entity_note_service.list_notes`/`add_note` don't check the parent `ClientRecord` exists — writes/reads against a non-existent client succeed silently instead of 404. | `app/notes/services/entity_note_service.py:38-67` | open |
| F-018 | Low | notes | `_assert_business_belongs_to_client` raises `BUSINESS.NOT_FOUND` with a business-flavored Hebrew message even when the client (not the business) is missing. Wrong code + misleading message. | `app/notes/services/business_note_service.py:85-88` | open |
| F-019 | Low (doc-only) | correspondence | `app/correspondence/README.md` documents `CORRESPONDENCE.FORBIDDEN_BUSINESS` as raised; code never raises it (business ownership uses `BUSINESS.NOT_FOUND` via `business_guards.py`). Stale README. | `app/correspondence/README.md:127` | open |
| F-020 | Medium | work-queue | Advance-payment rows in the queue use legacy `AdvancePayment.due_date` instead of `due_date_effective` — same INV-05 violation as F-005, just in the queue source builder. | `app/work_queue/services/billing_items.py:33,47` | open |
| F-021 | Low / design | work-queue | When `business_id` filter is set, all system source items (VAT/advance/binders/charges) are skipped entirely, so the queue scoped to a business shows tasks only. Filter-by-business silently hides system work. | `app/work_queue/services/work_queue_service.py:287-288` | open |

## Notes

- **F-001 is the priority.** Cross-report mutation via unchecked `line_id`. Suggested fix: scope repository update/delete by both `line_id` and `annual_report_id`, matching the annex service ownership check at `app/annual_reports/services/annex_service.py:87`.
- New findings from later domain runs append here. Keep each domain doc's `## Known issues` as the source detail; this table is the index.
