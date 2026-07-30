## Scope
This file owns only:
- Canonical current-state documentation for the annual-reports domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Annual Reports

The annual reports domain manages Israeli annual income-tax report work items per client record and tax year, including lifecycle status, statutory form/profile metadata, required schedules, annex data, income/expense lines, tax calculation, readiness checks, amendment history, PDF export, and client/year season views. Client approval/signature is not part of this workflow.
Last verified against code + backend/openapi.json: 2026-07-30 (W4 amendment branch).

## Endpoints

All paths below exist in `backend/openapi.json`. The `annual_reports` router is mounted under `/api/v1` in `backend/app/router_registry.py`; sub-routers are aggregated in `backend/app/annual_reports/api/annual_report_routers.py`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/annual-reports/{report_id}/annex/{schedule}` | List annex data lines for a report schedule. |
| `POST` | `/api/v1/annual-reports/{report_id}/annex/{schedule}` | Add an annex data line. |
| `PATCH` | `/api/v1/annual-reports/{report_id}/annex/{schedule}/{line_id}` | Update an annex data line. |
| `DELETE` | `/api/v1/annual-reports/{report_id}/annex/{schedule}/{line_id}` | Delete an annex data line. |
| `GET` | `/api/v1/annual-reports/{report_id}/details` | Get supplementary report detail fields. |
| `PATCH` | `/api/v1/annual-reports/{report_id}/details` | Update supplementary report detail fields. |
| `POST` | `/api/v1/annual-reports/tax-preview` | Calculate a pre-creation tax preview from supplied values. |
| `GET` | `/api/v1/annual-reports/{report_id}/financials` | Get income/expense lines and taxable-income summary. |
| `GET` | `/api/v1/annual-reports/{report_id}/tax-calculation` | Calculate tax, national insurance, advances, VAT balance, and liability summary. |
| `GET` | `/api/v1/annual-reports/{report_id}/advances-summary` | Get advance-payment summary scoped to the report client/year. |
| `GET` | `/api/v1/annual-reports/{report_id}/readiness` | Check filing readiness gates. |
| `PATCH` | `/api/v1/annual-reports/{report_id}` | Reassign the report (`assigned_to`) — the one report-level field editable outside the status flow. |
| `POST` | `/api/v1/annual-reports/{report_id}/income` | Add an income line. |
| `PATCH` | `/api/v1/annual-reports/{report_id}/income/{line_id}` | Update an income line. |
| `DELETE` | `/api/v1/annual-reports/{report_id}/income/{line_id}` | Delete an income line. |
| `POST` | `/api/v1/annual-reports/{report_id}/expenses` | Add an expense line. |
| `PATCH` | `/api/v1/annual-reports/{report_id}/expenses/{line_id}` | Update an expense line. |
| `DELETE` | `/api/v1/annual-reports/{report_id}/expenses/{line_id}` | Delete an expense line. |
| `POST` | `/api/v1/annual-reports/{report_id}/auto-populate` | Import VAT-derived income and expense lines. |
| `POST` | `/api/v1/annual-reports` | Create an annual report. |
| `GET` | `/api/v1/annual-reports` | List annual reports, optionally filtered by tax year. |
| `GET` | `/api/v1/annual-reports/overdue` | List open reports past filing deadline, returned as `AnnualReportListResponse` (`items`, `page`, `page_size`, `total`). |
| `GET` | `/api/v1/annual-reports/{report_id}` | Get full report detail. |
| `DELETE` | `/api/v1/annual-reports/{report_id}` | Soft-delete a report. |
| `POST` | `/api/v1/annual-reports/{report_id}/amend` | Advisor creates a new correction record from a submitted chain tip. |
| `POST` | `/api/v1/annual-reports/{report_id}/withdraw` | Advisor withdraws an unsubmitted correction and restores its original. |
| `GET` | `/api/v1/annual-reports/{report_id}/chain` | List the tax year's correction chain, oldest first, including withdrawn corrections. |
| `POST` | `/api/v1/annual-reports/{report_id}/schedules` | Add a schedule to a report. |
| `GET` | `/api/v1/annual-reports/{report_id}/schedules` | List schedules for a report. |
| `POST` | `/api/v1/annual-reports/{report_id}/schedules/complete` | Mark a schedule complete. |
| `POST` | `/api/v1/annual-reports/{report_id}/status` | Transition to a specific status. |
| `POST` | `/api/v1/annual-reports/{report_id}/submit` | Submit a report through the status transition path. |
| `POST` | `/api/v1/annual-reports/{report_id}/deadline` | Update deadline type and deadline note. |
| `GET` | `/api/v1/clients/{client_record_id}/annual-reports` | List annual reports for a client record. |
| `GET` | `/api/v1/tax-year/active/reports` | List reports for the active annual-report tax year. |
| `GET` | `/api/v1/tax-year/active/summary` | Summarize the active annual-report tax year. |
| `GET` | `/api/v1/tax-year/default` | Return the default annual-report tax year. |
| `GET` | `/api/v1/tax-year/{tax_year}/reports` | List reports for a tax year. |
| `GET` | `/api/v1/tax-year/{tax_year}/summary` | Summarize reports for a tax year. |
| `GET` | `/api/v1/annual-reports/{report_id}/export/pdf` | Download a draft annual-report PDF. |
| `POST` | `/api/v1/annual-reports/{report_id}/tax-calculation/save` | Persist `tax_due` or `refund_due`. |
| `GET` | `/api/v1/annual-reports/{report_id}/charges` | List charges linked to a report, returned as `PaginatedResponse[ChargeResponse]` (`items`, `page`, `page_size`, `total`). |

The PDF export is documented in OpenAPI as `application/pdf` with binary schema, not as an empty JSON response (`backend/app/annual_reports/api/annual_report_routes_export.py`). Integer path parameters for report IDs, line IDs, client record IDs, and tax-year route values publish positive bounds (`minimum: 1`) in OpenAPI (`backend/app/core/path_params.py`).

## Model & fields

Key models are:

- `AnnualReport` (`annual_reports`): root aggregate. Required fields: `id`, `client_record_id` FK to `client_records.id`, `tax_year`, `client_type`, `form_type`, shared `ObligationStatus`, `deadline_type`, `tax_calendar_entry_id` FK to `tax_calendar_entries.id`, `created_at`, `updated_at`; optional fields: `created_by`, `assigned_to`, `filing_deadline`, `custom_deadline_note`, `ita_reference`, `assessment_amount`, `refund_due`, `tax_due`, `submission_method`, `extension_reason`, `notes`, `deleted_at`, `deleted_by`; closing facts `closed_at`, `closed_by`, `closed_late`; amendment-chain facts from `AmendableMixin`: nullable self-FK `amends_id`, `superseded_at`, and `chain_closed_late`; non-null boolean flags default false: `has_rental_income`, `has_capital_gains`, `has_foreign_income`, `has_depreciation`, `has_exempt_rental`. There is no stored `is_amendment` flag: it is derived from `amends_id`. See `backend/app/annual_reports/models/annual_report_model.py:40-134`, `backend/app/common/obligation_chain.py:49-92`.
- The slot index is unique on `(client_record_id, tax_year)` only for rows that are not deleted, not amendments, and not canceled: `deleted_at IS NULL AND amends_id IS NULL AND status <> 'canceled'`. A partial unique index on `amends_id` prevents a chain from forking; the active tax-year/status index uses `superseded_at IS NULL`, matching normal chain-tip reads. See `backend/app/annual_reports/models/annual_report_model.py:136-170`.
- `AnnualReportDetail` (`annual_report_details`): 1:1 extension via unique non-null `report_id` FK; optional deduction/credit metadata (`pension_contribution`, `donation_amount`, `other_credits`), `internal_notes`, `amendment_reason`, nullable `updated_at`, and required `created_at`. `client_approved_at` retired with the signature flow (D-5, W4-pre). Derived credit points are not stored here. See `backend/app/annual_reports/models/annual_report_detail.py:13`.
- `AnnualReportIncomeLine` (`annual_report_income_lines`): `annual_report_id` FK, `source_type`, non-null `amount`, optional `description`, timestamps; includes a DB check `amount >= 0`. See `backend/app/annual_reports/models/annual_report_income_line.py:41`.
- `AnnualReportExpenseLine` (`annual_report_expense_lines`): `annual_report_id` FK, `category`, non-null `amount`, non-null `recognition_rate`, optional external/supporting document references and description. `supporting_document_id` is an optional FK to `permanent_documents.id`. See `backend/app/annual_reports/models/annual_report_expense_line.py:41`.
- `AnnualReportScheduleEntry` (`annual_report_schedules`): `annual_report_id` FK, `schedule`, required `is_required`, required `is_complete`, optional `notes`, `completed_at`, `completed_by`; unique per `(annual_report_id, schedule)`. See `backend/app/annual_reports/models/annual_report_schedule_entry.py:20`.
- `AnnualReportAnnexData` (`annual_report_annex_data`): `schedule_entry_id` FK, `line_number`, JSON `data`, `data_version`, optional `notes`, timestamps; unique per `(schedule_entry_id, line_number)`. See `backend/app/annual_reports/models/annual_report_annex_data.py:24`.
- Annual-report status history is stored in generic `EntityAuditLog` as `annual_report.status_changed`. The legacy `AnnualReportStatusHistory` model/table and per-domain `GET /annual-reports/{report_id}/audit` route are removed; Phase 9 also moved demo seed status history to `EntityAuditWriter`.
- `AnnualReportCreditPoint` (`annual_report_credit_points`): `annual_report_id` FK, `reason`, non-null `points`, optional `notes`, unique per `(annual_report_id, reason)`. See `backend/app/annual_reports/models/annual_report_credit_point_reason.py:21`.

## Enums / statuses

- `ClientAnnualFilingType`: `individual`, `self_employed`, `corporation`, `public_institution`, `partnership`, `control_holder`, `exempt_dealer`. See `backend/app/annual_reports/models/annual_report_enums.py:8`.
- `PrimaryAnnualReportForm`: `1301`, `1214`, `1215`. See `backend/app/annual_reports/models/annual_report_enums.py:27`.
- Annual reports have no domain-local status enum. They use shared `ObligationStatus`: `awaiting_input`, `input_received`, `in_progress`, `awaiting_verification`, `submitted`, `canceled`. See `backend/app/common/enums.py` and `backend/app/common/obligation_lifecycle.py`.
- `AnnualReportSchedule`: `schedule_a`, `schedule_b`, `schedule_gimmel`, `schedule_dalet`, `form_150`, `form_1504`, `form_6111`, `form_1344`, `form_1399`, `form_1350`, `form_1327`, `form_1342`, `form_1343`, `form_1348`, `form_858`. See `backend/app/annual_reports/models/annual_report_enums.py:45`.
- `FilingDeadlineType`: `standard`, `extended`, `custom`. See `backend/app/annual_reports/models/annual_report_enums.py:65`.
- `ExtensionReason`: `military_service`, `health_reason`, `general`, `war_situation`. See `backend/app/annual_reports/models/annual_report_enums.py:84`.
- `IncomeSourceType`: `business`, `salary`, `interest`, `dividends`, `capital_gains`, `rental`, `foreign`, `pension`, `other`. See `backend/app/annual_reports/models/annual_report_income_line.py:22`.
- `ExpenseCategoryType`: `office_rent`, `professional_services`, `salaries`, `depreciation`, `vehicle`, `marketing`, `insurance`, `communication`, `travel`, `training`, `bank_fees`, `other`. See `backend/app/annual_reports/models/annual_report_expense_line.py:21`. **Domain split:** these 12 values are higher-level recognition categories for income-tax reporting, distinct from VAT's `ExpenseCategory` (24 granular data-entry values). VAT categories are mapped into these via `_VAT_TO_ANNUAL` in `backend/app/annual_reports/services/annual_report_vat_import_service.py`. See `docs/domains/vat.md` for the VAT side.
- `CreditPointReason`: `resident`, `academic_degree`, `discharged_soldier`, `new_immigrant`, `single_parent`. See `backend/app/annual_reports/models/annual_report_credit_point_reason.py:13`.

The shared graph permits forward movement one stage at a time, backward movement one stage with a reason, cancellation from an unlocked stage, and no transition out of `submitted` or `canceled`. `ReportStage`, its lossy shortcut map, and `POST /annual-reports/{id}/transition` were removed in W2.


## Lifecycle

**This domain runs the shared obligation lifecycle.** VAT, advance payments and
annual reports are three views of one thing — a client owes an obligation for a
period, works it through, and settles it by a deadline — and they now share one
status enum and one transition graph:

| # | `ObligationStatus` | Label |
|---|---|---|
| 1 | `awaiting_input` | ממתין לחומר |
| 2 | `input_received` | החומר התקבל |
| 3 | `in_progress` | בעבודה |
| 4 | `awaiting_verification` | ממתין לאימות |
| 5 | `submitted` | הוגש |
| — | `canceled` | בוטל (off-ladder, reachable from any unlocked stage) |

Rules, enforced once in `app/common/obligation_lifecycle.py`:

- **Forward one stage at a time.** An event may perform consecutive transitions and
  records each; it never skips a stage's meaning.
- **Backward one stage at a time, always with a reason** (`OBLIGATION.TRANSITION_REASON_REQUIRED`).
- **`submitted` has no outgoing transition** (`OBLIGATION.LOCKED`). Correcting a
  submitted obligation creates a separate amendment record through `POST /amend`.
- **Cancel from any unlocked stage**, and `canceled` is terminal.

`RESOLVED_OBLIGATION_STATUSES` = `{submitted, canceled}` is the single answer to
"does this need further work?", read by both the Python predicate and every SQL
query. `OBLIGATION_STATUS_LABELS` is the single Hebrew vocabulary.

## Domain rules & invariants

- Reports are scoped to `client_record_id`, not a business. List and count queries filter by `client_record_id` and exclude soft-deleted rows. See `backend/app/annual_reports/repositories/annual_report_report_repository.py`.
- Creation asks the slot-occupancy query, whose predicate matches the partial unique index: a canceled original frees the tax year, an amendment never occupies it, and a superseded original continues to occupy it. A duplicate slot raises `ANNUAL_REPORT.CONFLICT`. See `backend/app/annual_reports/services/annual_report_create_service.py:83-95`, `backend/app/annual_reports/repositories/annual_report_report_repository.py:126`.
- Creation requires an existing active client record, valid `client_type`, valid `deadline_type`, and an existing assignee when `assigned_to` is supplied. See `backend/app/annual_reports/services/annual_report_create_service.py`.
- Creation derives `form_type` from `client_type`, calculates the filing deadline, ensures and links an annual tax-calendar entry, generates initial schedules, and writes `annual_report.created` to generic `EntityAuditLog`. See `backend/app/annual_reports/services/annual_report_create_service.py`.
- Primary forms are mapped as: `individual`, `self_employed`, `partnership`, `control_holder`, `exempt_dealer` -> `1301`; `corporation` -> `1214`; `public_institution` -> `1215`. Form `0135` is explicitly out of the full annual-return workflow. See `backend/app/annual_reports/annual_report_constants.py`.
- Standard deadlines use tax-rules registry data when available; corporate, public-institution, and control-holder profiles use corporate-style deadlines; online/representative individual-style submissions use June 30; fallback manual individual-style deadline is May 29. Extended deadline is January 31 of `tax_year + 2`. See `backend/app/annual_reports/annual_report_deadlines.py`.
- Initial required schedules are auto-generated: self-employed and partnership get `schedule_a`; partnership also gets `form_1504`; `has_rental_income`, `has_capital_gains`, and `has_foreign_income` add `schedule_b`, `schedule_gimmel`, and `schedule_dalet`. See `backend/app/annual_reports/services/annual_report_schedule_service.py`.
- Status transitions lock the report row, validate the target `ObligationStatus`, enforce the shared transition graph, and write `annual_report.status_changed` to generic `EntityAuditLog` with `client_record_id`, `tax_year`, old/new status, note, actor id, and actor display-name snapshot. A failed audit write rolls back the status mutation. See `backend/app/annual_reports/services/annual_report_status_service.py`.
- Transitioning to `submitted` always runs the readiness check first, and records the closing facts: `closed_by` (the acting user), `closed_at`, and `closed_late` — the Israel-local date of `closed_at` against the deadline as recalculated by the submission itself, NULL when there is no deadline (D-32). See `backend/app/annual_reports/services/annual_report_status_service.py`.
- **A submitted report is fully locked (D-13, W3):** financial lines, detail, schedules/annexes, credit points, deadline, tax-calculation save, reassignment, and soft delete are all rejected with `OBLIGATION.LOCKED` (`assert_report_unlocked`, `backend/app/annual_reports/annual_report_financial_line_helpers.py`). The financial-mutation guard answers two different questions and both run: the client's standing (closed/frozen client) *and* the report's own lifecycle. Every change to a submitted report goes through an amendment (W4).
- Readiness has four gates: an assignee set (the shared closing gate, D-15/W3), required schedules complete, total income greater than zero, and either `tax_due` or `refund_due` persisted. Completion percent is `passed / 4 * 100`. The client-approval gate retired with the signature flow (D-5, W4-pre). See `backend/app/annual_reports/services/annual_report_readiness_service.py`.
- **No client signature or client review on an annual report (D-5, W4-pre).** The office reviews and files alone. `awaiting_verification` means *ready, awaiting internal review* — the same meaning it carries in VAT — and entering it has no side-effects: no signature request, no client email, no external signing page. Only an advisor moves the report from there to `submitted`, so a submitted annual report always names its closer (`closed_by` is never NULL).
- Updating a deadline recalculates `filing_deadline` for `standard`/`extended`, sets `None` for `custom`, and writes `annual_report.deadline_updated` to generic `EntityAuditLog`. See `backend/app/annual_reports/services/annual_report_status_service.py`.
- `POST /annual-reports/{id}/amend` is advisor-only and accepts only a submitted chain tip. It locks the original and creates a new report at `in_progress`. The copy includes the detail row, income and expense lines, credit points, schedules, and schedule annex lines; it clears identity, soft-delete state, closing/submission facts, filing deadline/custom-deadline details, assessment, and persisted `tax_due`/`refund_due`. It keeps the same `tax_calendar_entry_id`, links through `amends_id`, stamps the original's `superseded_at`, and carries `chain_closed_late`. The old same-row `amend_report` reopen mechanism is retired. See `backend/app/annual_reports/services/annual_report_amendment_service.py`, `backend/app/annual_reports/repositories/annual_report_report_repository.py:47-79`.
- Normal lists, counts, sums, VAT imports, and season summaries use the chain-tip scope (`deleted_at IS NULL AND superseded_at IS NULL`); `GET /chain` is the explicit history read.
- Plain DELETE of an amendment returns 409 `OBLIGATION.AMENDMENT_NOT_DELETABLE`. `POST /annual-reports/{id}/withdraw` accepts an amendment that is not submitted and is still the chain tip, soft-deletes it, clears the original's `superseded_at`, returns the restored original, and leaves the withdrawn correction visible only in the chain response with `is_withdrawn=true`. A submitted correction is corrected by another amendment, not withdrawn.
- Financial summary totals income and recognized expenses; `taxable_income = total_income - recognized_expenses`. The detail response (`AnnualReportDetailResponse`) groups every computed financial/tax output under a nested `tax_calculation` object (`AnnualReportTaxCalculationResponse`): `total_income`, `total_expenses` (gross), `recognized_expenses`, `taxable_income`, `profit`, `tax_after_credits`, `final_balance` (after subtracting paid advances), and the aggregated credit-point breakdown (`credit_points`, `pension_credit_points`, `life_insurance_credit_points`, `tuition_credit_points`). User-entered deduction inputs (`pension_contribution`, `donation_amount`, `other_credits`) and the persisted outcome columns (`assessment_amount`, `refund_due`, `tax_due`) stay flat on the detail response. See `backend/app/annual_reports/services/annual_report_financial_summary_service.py` and `backend/app/annual_reports/services/annual_report_query_service.py`.
- List endpoints (`GET /annual-reports`, `/overdue`, season and client report lists) return the thin `AnnualReportListItem` row DTO (identity, status, deadlines, the persisted outcome amounts, and `amends_id`); they intentionally omit detail/calculation/action/transition fields. Only `GET /annual-reports/{id}` returns the full `AnnualReportDetailResponse`. `amends_id` is not a figure the list renders: a chain shows as one row everywhere (D-12), so without it a correction and the report it replaced are indistinguishable in every cell a list row has. See `backend/app/annual_reports/services/annual_report_base_service.py` (`_to_list_items`).
- Tax calculation is read-only. It uses financial summary, detail deductions/credits, credit-point rows/default resident points, income-tax engine, national-insurance engine, VAT net balance, and paid advances. `tax_after_credits` is the computed tax before advances; `final_balance = tax_after_credits - advances_paid` (negative = refund expected).
- **`advances_paid` and `final_balance` have one definition, published by `AnnualReportTaxService`** on `TaxCalculationResponse`. Both consumers — the detail response's `tax_calculation` and the advances summary — read those fields; neither recomputes the subtraction, and neither reads advances from anywhere but `AdvancePaymentAggregationRepository.sum_paid_by_client_year`. The advances summary previously summed `paid_amount` in Python over a `page_size=10000` read, which silently undercounted past the cap and could disagree with the detail response for the same report. Public tax calculation amounts, rates, bracket rates, and credit-point totals are serialized as `ApiDecimal` strings; frontend code parses them only for display/math. See `backend/app/annual_reports/services/annual_report_tax_service.py`.
- Persisting a tax calculation rejects requests with both `tax_due` and `refund_due`. See `backend/app/annual_reports/services/annual_report_tax_service.py`.
- Income and expense line create/update/delete are financial mutations. A missing or soft-deleted related `ClientRecord` raises `CLIENT_RECORD.NOT_FOUND`; a frozen or closed client raises `CLIENT_RECORD.CLOSED` before creating, updating, or deleting annual-report income/expense lines. Successful manual income/expense mutations clear saved `tax_due` and `refund_due` while the report is pre-submission (`awaiting_input`, `input_received`, `in_progress`, or `awaiting_verification`). See `backend/app/annual_reports/annual_report_financial_line_helpers.py`, `backend/app/clients/guards/client_record_guards.py`, and `backend/app/annual_reports/services/annual_report_financial_line_service.py`.
- Income/expense source/category values are validated against enum values; audit entries are written for manual line mutations. See `backend/app/annual_reports/services/annual_report_financial_line_service.py`.
- VAT auto-populate is a financial mutation and follows the same mutation rules as manual income/expense line changes. It requires a concrete actor id so audit cannot be silently skipped, raises `CLIENT_RECORD.NOT_FOUND` when the related `ClientRecord` is missing or soft-deleted, and is blocked for closed/frozen clients, including `force=True`; it writes audit records for created income lines, created expense lines, and deleted/replaced lines when `force=True`. Created-line audit payloads mark `source=vat_import`; force-replacement delete payloads mark `mutation_source=vat_import` and `mutation_reason=force_replace`. See `backend/app/annual_reports/services/annual_report_vat_import_service.py`.
- VAT auto-populate is allowed only in `awaiting_input`, `input_received`, and `in_progress`; existing income/expense lines cause `ANNUAL_REPORT.LINES_ALREADY_EXIST` unless `force=True`; import aggregates VAT chain tips by `client_record_id` and `tax_year`, maps VAT categories to annual expense categories, and writes lines only for positive generated totals. See `backend/app/annual_reports/services/annual_report_vat_import_service.py:53-58`.
- VAT auto-populate response includes `skipped_items`, `warnings`, and `expense_breakdown`. `skipped_items` includes both true skipped totals and review items that need user attention. Zero generated expense category totals are returned as skipped items and do not create financial lines; zero income or no VAT data creates no line and no skipped noise. Negative generated totals are returned as skipped items with warnings and do not create financial lines automatically. A negative VAT source category inside a positive merged annual-report category is returned as `negative_source_contribution` with a warning while the positive net annual category can still create a line. `expense_breakdown` is the VAT source merge breakdown and includes both imported and skipped merged expense categories. Merged annual-report categories return `source_vat_categories`, and VAT-import expense audit payloads store the same source breakdown. See `backend/app/annual_reports/schemas/annual_report_financials.py` and `backend/app/annual_reports/services/annual_report_vat_import_service.py`.
- Client freezing/closing integration cancels open annual reports by directly setting non-terminal reports to `canceled`. See `backend/app/annual_reports/repositories/annual_report_report_repository.py`.

Future / planned:

- Dedicated financial-history endpoints for client annual-report comparisons are not implemented. See `docs/archive/annual-reports-legacy.md`.

## Error codes

Registry: `docs/backend/error-codes.md`.

The annual reports namespace is registered as `ANNUAL_REPORT` in `docs/backend/error-codes.md:43`. Codes raised by this module include:

- `ANNUAL_REPORT.NOT_FOUND`
- `ANNUAL_REPORT.INVALID_STATUS`
- `OBLIGATION.LOCKED` — any mutation on a submitted report (W3 full lock)
- `ANNUAL_REPORT.INVALID_TYPE`
- `ANNUAL_REPORT.CLIENT_NOT_FOUND`
- `ANNUAL_REPORT.CONFLICT`
- `ANNUAL_REPORT.LINE_NOT_FOUND`
- `ANNUAL_REPORT.ANNEX_VALIDATION_ERROR`
- `ANNUAL_REPORT.TAX_CONFLICT`
- `ANNUAL_REPORT.INVALID_STATUS_FOR_AUTOPOPULATE`
- `ANNUAL_REPORT.LINES_ALREADY_EXIST`
- `ANNUAL_REPORT.AUDIT_ACTOR_REQUIRED`

Related codes raised from this module but owned by another namespace:

- `CLIENT_RECORD.NOT_FOUND` when a related client record is missing or soft-deleted during financial mutation validation.
- `CLIENT_RECORD.CLOSED` when a financial mutation is attempted for a frozen or closed client record.
- `OBLIGATION.NOT_CLOSED` when amendment creation is requested from a record that is not submitted.
- `OBLIGATION.ALREADY_AMENDED` when the record already has a correction or withdrawal targets a non-tip link.
- `OBLIGATION.AMENDMENT_NOT_DELETABLE` when plain DELETE is requested for an amendment.
- `OBLIGATION.NOT_AN_AMENDMENT` when withdrawal is requested for a standalone record.

Source grep: `backend/app/annual_reports/services/*`.

## Known issues

Reconciled against `backend/app/annual_reports/README.md` on 2026-07-27; every item below was
re-verified against code. Items that README still listed as open, but code has since resolved, were
moved to Resolved issues rather than carried over.

- **VAT balance is summed into income-tax liability.** `total_liability` is computed as
  `tax_after_credits + ni.total + (vat_balance or 0) - advances_paid`, mixing an income-tax figure,
  a VAT balance, and an advances credit into one number. The PDF renders that same value labelled
  `סה"כ חבות (מס + ביטוח לאומי)`, which does not mention VAT — so the label contradicts the
  arithmetic. Violates the invariant that a domain's published total means what its label says.
  Suggested fix: drop `vat_balance` from the sum and expose it as a separate informational field.
  Cite: `backend/app/annual_reports/services/annual_report_tax_service.py:127-129`,
  `backend/app/annual_reports/annual_report_pdf_builder.py:265`.

- **Persisted `tax_due` / `refund_due` can be stale, and readiness reads them.** Manual
  income/expense mutations invalidate the persisted result, but `VatImportService.auto_populate`
  creates and deletes financial lines through repositories directly and does not. The readiness gate
  then reads the stale persisted value. Violates "persisted tax results reflect current financial
  lines and current advances". Suggested fix: move invalidation into the owning services so every
  path that changes an input triggers it. Cite:
  `backend/app/annual_reports/services/annual_report_vat_import_service.py`,
  `backend/app/annual_reports/services/annual_report_tax_service.py:179`.

  The `AdvancePaymentService.bulk_mark_paid` half of this issue is **resolved** — it invalidates at
  `advance_payment_service.py:508`.

- **`AnnualReportDetail.updated_at` is nullable.** Should be `nullable=False` with a default and a
  backfill migration. Cite: `backend/app/annual_reports/models/annual_report_detail.py:57`.

Unverified external tax constants — these need checking against the authority's published circulars,
which cannot be confirmed from code:

- **2026 national-insurance ceiling** is `622_920` in `ni_engine.py`; verify against the 2026 NII circular.
- **2026 tax brackets** in `tax_engine.py`: the 5th-bracket ceiling appears lower than 2025; verify against the 2026 ITA circular.
- **`DONATION_MINIMUM_ILS = 190`**; verify against the current ITA Section 46 indexed amount (may be 180 for 2024).

## Resolved issues

- **Stale README items reconciled** (2026-07-27): four items still listed as open in
  `backend/app/annual_reports/README.md` were verified as already fixed in code and are recorded
  here rather than as known issues. Two of those fixes were later superseded by deletion:
  (1) the annual-report signature flow retired entirely in W4-pre, so the old business-scoping
  failure cannot occur; (2) the same-row `amend_report` command retired and W4 replaced it with
  `POST /annual-reports/{id}/amend`, which creates a separate record. The other two remain:
  (3) `business_name` does not appear in any annual-report schema or service; (4)
  `AnnualReportAnnexData` has `UniqueConstraint(schedule_entry_id, line_number)`; because
  `annual_report_schedules` is already unique on `(annual_report_id, schedule)`, this is equivalent
  to the per-report-per-schedule uniqueness the README asked for.

- **MAT-54** (2026-07-23, **dissolved 2026-07-29**): Signing an annual-report approval could finish as `signed` without submitting the report. Fixed at the time by persisting the approval timestamp first, isolating the transition in a savepoint, and reconciling. The whole mechanism retired with D-5 (W4-pre) — there is no annual-report signature to reconcile.
- **F-001** (2026-06-04): Income/expense line update and delete paths checked only that the report existed, not that the line belonged to it. Fixed: repository mutation and audit snapshots are scoped by both `line_id` and `annual_report_id`.
- **F-002 / F-AR-001** (2026-06-04, **dissolved 2026-07-29**): Transitioning to `pending_client` silently skipped signature creation. Fixed at the time with pre-write validation; the transition and its signature side-effect both retired with D-5 (W4-pre).
- **F-003** (2026-06-04): Legacy docs described annual-report status transitions syncing linked tax-calendar entries and reminders. Retired as stale: `TaxCalendarEntry` has no status field; grouped calendar reads `AnnualReport.status` live; reminders domain has no coupling to annual-reports.
- **F-004** (2026-06-04): VAT auto-populate was flagged for aggregating by `client_record_id`+`tax_year` with no per-business selector. Accepted design: annual reports are client-scoped; Business is activity grouping only. Adding `business_id` filter would create a client-level report with a business-level import boundary — an inconsistency. See `backend/app/annual_reports/services/annual_report_vat_import_service.py`.
- **F-AR-035/036/037** (2026-06-11): `AnnualReportDetailResponse` schema cleanup (api-todo 35-37). (1) Removed duplicate float copies `tax_refund_amount`/`tax_due_amount` — investigation confirmed they were unrendered copies of the canonical DB columns `refund_due`/`tax_due`. The premise's other two "pairs" were not duplicates: `assessment_amount` is the external ITA assessment input; `final_balance` is computed (`tax_after_credits − advances_paid`). (2) Grouped all computed financial/tax outputs under nested `tax_calculation`. (3) Split list and detail DTOs: list endpoints now return the thin `AnnualReportListItem`, detail keeps `AnnualReportDetailResponse`. No backwards-compat shims (per entry-point.md). Canonical choices: `refund_due` over `tax_refund_amount`, `tax_due` over `tax_due_amount`.

## Decisions (preserved)

- The domain models full annual returns, not short refund-request form `0135`; primary form selection is controlled by `ClientAnnualFilingType -> PrimaryAnnualReportForm`. This is implemented in `FORM_MAP` and preserved from legacy summary context. See `backend/app/annual_reports/annual_report_constants.py`.
- `6111` is modeled as an annex/schedule rather than a primary return. This is implemented as `form_6111` in `AnnualReportSchedule`. See `backend/app/annual_reports/models/annual_report_enums.py:45`.
- Annual reports are owned by client record and tax year, not by business. The root model, list queries, and VAT/advance/tax calculations use `client_record_id` and `tax_year`. See `backend/app/annual_reports/models/annual_report_model.py:43` and `backend/app/annual_reports/services/annual_report_tax_service.py`.
- Annual-report financial line mutations fail closed when the related client record is missing or soft-deleted, and are locked for closed/frozen clients. This includes manual income/expense create, update, delete, and VAT auto-populate, including forced replacement. Manual income/expense mutations also clear persisted tax results before submission so readiness cannot rely on stale `tax_due` or `refund_due`.
- VAT auto-populate is a traceable financial mutation. It requires an actor id, audits created lines and deleted/replaced lines, marks the mutation source as VAT import, skips zero generated expense totals without creating lines, keeps empty zero income quiet, returns negative totals as skipped items/warnings, surfaces negative source contributions inside positive merged categories, and returns source VAT-category breakdown for merged annual-report categories.
- Annual reports carry a required `tax_calendar_entry_id` FK. Creation ensures an annual tax-calendar entry and links the report to it. This preserves the still-implemented tax-calendar decision from `backend/docs/domain_decisions_v3.md`. See `backend/app/annual_reports/models/annual_report_model.py` and `backend/app/annual_reports/services/annual_report_create_service.py`.
- VAT auto-populate is intentionally client-wide. It aggregates all VAT work items for the report's `client_record_id`+`tax_year` across all businesses. Annual reports are client-scoped; Business is activity grouping only and is not the accounting boundary. Adding `business_id` filtering to auto_populate would create a client-level report with a business-level import boundary — an inconsistency. Do not add a `business_id` param. See `backend/app/annual_reports/services/annual_report_vat_import_service.py` and `test_vat_auto_populate_aggregates_all_businesses_for_client_year`.
- Report-history list endpoints stay light: `GET /api/v1/clients/{client_record_id}/annual-reports` returns `AnnualReportListResponse`, and there is no current dedicated financial-history endpoint. This preserves the legacy product decision discussion while marking financial history as future/planned.
- Historical external government/legal sources are archived for reference only. They are not canonical implementation behavior and should not override code or architecture docs.

## Future / planned

- Add a dedicated client annual-report financial history endpoint, if the product wants a multi-year financial comparison table. Candidate paths discussed historically: `GET /api/v1/clients/{client_record_id}/annual-reports/history` or `GET /api/v1/annual-reports/{report_id}/client-history`. These do not exist in `backend/openapi.json` as of 2026-05-29.

## Historical notes

Archived legacy material: `docs/archive/annual-reports-legacy.md`.
