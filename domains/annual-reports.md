## Scope
This file owns only:
- Canonical current-state documentation for the annual-reports domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Annual Reports

The annual reports domain manages Israeli annual income-tax report work items per client record and tax year, including lifecycle status, statutory form/profile metadata, required schedules, annex data, income/expense lines, tax calculation, readiness checks, client approval, PDF export, and client/year season views.
Last verified against code + backend/openapi.json: 2026-06-04.

## Endpoints

All paths below exist in `backend/openapi.json`. The `annual_reports` router is mounted under `/api/v1` in `backend/app/router_registry.py:33`; sub-routers are aggregated in `backend/app/annual_reports/api/routers.py:22`.

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
| `POST` | `/api/v1/annual-reports/{report_id}/amend` | Reopen a submitted report for amendment. |
| `POST` | `/api/v1/annual-reports/{report_id}/schedules` | Add a schedule to a report. |
| `GET` | `/api/v1/annual-reports/{report_id}/schedules` | List schedules for a report. |
| `POST` | `/api/v1/annual-reports/{report_id}/schedules/complete` | Mark a schedule complete. |
| `POST` | `/api/v1/annual-reports/{report_id}/transition` | Move by high-level stage shortcut. |
| `POST` | `/api/v1/annual-reports/{report_id}/status` | Transition to a specific status. |
| `POST` | `/api/v1/annual-reports/{report_id}/submit` | Submit a report through the status transition path. |
| `POST` | `/api/v1/annual-reports/{report_id}/deadline` | Update deadline type and deadline note. |
| `GET` | `/api/v1/annual-reports/{report_id}/audit` | List status audit entries. |
| `GET` | `/api/v1/clients/{client_record_id}/annual-reports` | List annual reports for a client record. |
| `GET` | `/api/v1/tax-year/active/reports` | List reports for the active annual-report tax year. |
| `GET` | `/api/v1/tax-year/active/summary` | Summarize the active annual-report tax year. |
| `GET` | `/api/v1/tax-year/default` | Return the default annual-report tax year. |
| `GET` | `/api/v1/tax-year/{tax_year}/reports` | List reports for a tax year. |
| `GET` | `/api/v1/tax-year/{tax_year}/summary` | Summarize reports for a tax year. |
| `GET` | `/api/v1/annual-reports/{report_id}/export/pdf` | Download a draft annual-report PDF. |
| `POST` | `/api/v1/annual-reports/{report_id}/tax-calculation/save` | Persist `tax_due` or `refund_due`. |
| `GET` | `/api/v1/annual-reports/{report_id}/charges` | List charges linked to a report. |

## Model & fields

Key models are:

- `AnnualReport` (`annual_reports`): root aggregate. Required fields: `id`, `client_record_id` FK to `client_records.id`, `tax_year`, `client_type`, `form_type`, `status`, `deadline_type`, `tax_calendar_entry_id` FK to `tax_calendar_entries.id`, `created_at`, `updated_at`; optional fields: `created_by`, `assigned_to`, `filing_deadline`, `custom_deadline_note`, `submitted_at`, `ita_reference`, `assessment_amount`, `refund_due`, `tax_due`, `submission_method`, `extension_reason`, `notes`, `deleted_at`, `deleted_by`; non-null boolean flags default false: `has_rental_income`, `has_capital_gains`, `has_foreign_income`, `has_depreciation`, `has_exempt_rental`. See `backend/app/annual_reports/models/annual_report_model.py:42`.
- `AnnualReport` has a partial unique active index on `(client_record_id, tax_year)` where `deleted_at IS NULL`, plus status, tax-year/status, calendar-entry, deadline, and assignee indexes. See `backend/app/annual_reports/models/annual_report_model.py:125`.
- `AnnualReportDetail` (`annual_report_details`): 1:1 extension via unique non-null `report_id` FK; optional deduction/credit metadata (`pension_contribution`, `donation_amount`, `other_credits`), `client_approved_at`, `internal_notes`, `amendment_reason`, nullable `updated_at`, and required `created_at`. Derived credit points are not stored here. See `backend/app/annual_reports/models/annual_report_detail.py:13`.
- `AnnualReportIncomeLine` (`annual_report_income_lines`): `annual_report_id` FK, `source_type`, non-null `amount`, optional `description`, timestamps; includes a DB check `amount >= 0`. See `backend/app/annual_reports/models/annual_report_income_line.py:41`.
- `AnnualReportExpenseLine` (`annual_report_expense_lines`): `annual_report_id` FK, `category`, non-null `amount`, non-null `recognition_rate`, optional external/supporting document references and description. `supporting_document_id` is an optional FK to `permanent_documents.id`. See `backend/app/annual_reports/models/annual_report_expense_line.py:41`.
- `AnnualReportScheduleEntry` (`annual_report_schedules`): `annual_report_id` FK, `schedule`, required `is_required`, required `is_complete`, optional `notes`, `completed_at`, `completed_by`; unique per `(annual_report_id, schedule)`. See `backend/app/annual_reports/models/annual_report_schedule_entry.py:20`.
- `AnnualReportAnnexData` (`annual_report_annex_data`): `schedule_entry_id` FK, `line_number`, JSON `data`, `data_version`, optional `notes`, timestamps; unique per `(schedule_entry_id, line_number)`. See `backend/app/annual_reports/models/annual_report_annex_data.py:24`.
- `AnnualReportStatusHistory` (`annual_report_status_history`): append-only status history with `annual_report_id`, optional `from_status`, required `to_status`, required `changed_by` FK to `users.id`, optional `note`, and required `occurred_at`. See `backend/app/annual_reports/models/annual_report_status_history.py:16`.
- `AnnualReportCreditPoint` (`annual_report_credit_points`): `annual_report_id` FK, `reason`, non-null `points`, optional `notes`, unique per `(annual_report_id, reason)`. See `backend/app/annual_reports/models/annual_report_credit_point_reason.py:21`.

## Enums / statuses

- `ClientAnnualFilingType`: `individual`, `self_employed`, `corporation`, `public_institution`, `partnership`, `control_holder`, `exempt_dealer`. See `backend/app/annual_reports/models/annual_report_enums.py:8`.
- `PrimaryAnnualReportForm`: `1301`, `1214`, `1215`. See `backend/app/annual_reports/models/annual_report_enums.py:27`.
- `AnnualReportStatus`: `not_started`, `collecting_docs`, `in_preparation`, `pending_client`, `submitted`, `closed`, `canceled`. See `backend/app/annual_reports/models/annual_report_enums.py:35`.
- `AnnualReportSchedule`: `schedule_a`, `schedule_b`, `schedule_gimmel`, `schedule_dalet`, `form_150`, `form_1504`, `form_6111`, `form_1344`, `form_1399`, `form_1350`, `form_1327`, `form_1342`, `form_1343`, `form_1348`, `form_858`. See `backend/app/annual_reports/models/annual_report_enums.py:45`.
- `FilingDeadlineType`: `standard`, `extended`, `custom`. See `backend/app/annual_reports/models/annual_report_enums.py:65`.
- `ReportStage`: `material_collection`, `in_progress`, `final_review`, `client_signature`, `transmitted`, `post_submission`. The implemented shortcut map does not include `post_submission`. See `backend/app/annual_reports/models/annual_report_enums.py:71` and `backend/app/annual_reports/services/constants.py:50`.
- `ExtensionReason`: `military_service`, `health_reason`, `general`, `war_situation`. See `backend/app/annual_reports/models/annual_report_enums.py:84`.
- `IncomeSourceType`: `business`, `salary`, `interest`, `dividends`, `capital_gains`, `rental`, `foreign`, `pension`, `other`. See `backend/app/annual_reports/models/annual_report_income_line.py:22`.
- `ExpenseCategoryType`: `office_rent`, `professional_services`, `salaries`, `depreciation`, `vehicle`, `marketing`, `insurance`, `communication`, `travel`, `training`, `bank_fees`, `other`. See `backend/app/annual_reports/models/annual_report_expense_line.py:21`.
- `CreditPointReason`: `resident`, `academic_degree`, `discharged_soldier`, `new_immigrant`, `single_parent`. See `backend/app/annual_reports/models/annual_report_credit_point_reason.py:13`.

Implemented status transitions are:

| From | Allowed to |
|------|------------|
| `not_started` | `collecting_docs` |
| `collecting_docs` | `in_preparation`, `not_started` |
| `in_preparation` | `pending_client`, `collecting_docs` |
| `pending_client` | `in_preparation`, `submitted` |
| `submitted` | `in_preparation`, `closed` |
| `closed` | none |
| `canceled` | none |

Source: `backend/app/annual_reports/services/constants.py:24`.

## Domain rules & invariants

- Reports are scoped to `client_record_id`, not a business. List and count queries filter by `client_record_id` and exclude soft-deleted rows. See `backend/app/annual_reports/repositories/report_repository.py:40`.
- A non-deleted client/year can have only one annual report. The service checks `get_by_client_record_year` and raises `ANNUAL_REPORT.CONFLICT`; the model also has a partial unique index. See `backend/app/annual_reports/services/create_service.py:83` and `backend/app/annual_reports/models/annual_report_model.py:125`.
- Creation requires an existing active client record, valid `client_type`, valid `deadline_type`, and an existing assignee when `assigned_to` is supplied. See `backend/app/annual_reports/services/create_service.py:53`.
- Creation derives `form_type` from `client_type`, calculates the filing deadline, ensures and links an annual tax-calendar entry, generates initial schedules, appends initial status history, and writes an audit create record. See `backend/app/annual_reports/services/create_service.py:95`.
- Primary forms are mapped as: `individual`, `self_employed`, `partnership`, `control_holder`, `exempt_dealer` -> `1301`; `corporation` -> `1214`; `public_institution` -> `1215`. Form `0135` is explicitly out of the full annual-return workflow. See `backend/app/annual_reports/services/constants.py:8`.
- Standard deadlines use tax-rules registry data when available; corporate, public-institution, and control-holder profiles use corporate-style deadlines; online/representative individual-style submissions use June 30; fallback manual individual-style deadline is May 29. Extended deadline is January 31 of `tax_year + 2`. See `backend/app/annual_reports/services/deadlines.py:44`.
- Initial required schedules are auto-generated: self-employed and partnership get `schedule_a`; partnership also gets `form_1504`; `has_rental_income`, `has_capital_gains`, and `has_foreign_income` add `schedule_b`, `schedule_gimmel`, and `schedule_dalet`. See `backend/app/annual_reports/services/schedule_service.py:45`.
- Status transitions lock the report row, validate the target status, enforce `VALID_TRANSITIONS`, append status history, and write an audit status-change record. See `backend/app/annual_reports/services/status_service.py:60`.
- Transitioning to `submitted` always runs the readiness check first. See `backend/app/annual_reports/services/status_service.py:116`.
- Readiness has four gates: required schedules complete, total income greater than zero, either `tax_due` or `refund_due` persisted, and `AnnualReportDetail.client_approved_at` set. Completion percent is `passed / 4 * 100`. See `backend/app/annual_reports/services/readiness_service.py`.
- Entering `pending_client` first validates the client record exists and that a signer name is resolvable (raises `CLIENT_RECORD.NOT_FOUND` or `ANNUAL_REPORT.SIGNER_NAME_MISSING` before any DB write), then cancels existing pending signature requests and creates a new annual-report signature request with `business_id=None`; signer name comes from `Person.full_name` (OWNER link) or `LegalEntity.official_name`. Leaving `pending_client` cancels pending signature requests. See `backend/app/annual_reports/services/status_service.py:162` and `status_signature_helper.py`.
- Updating a deadline recalculates `filing_deadline` for `standard`/`extended`, sets `None` for `custom`, appends a same-status history row, and writes an audit entry. See `backend/app/annual_reports/services/status_service.py:195`.
- `amend_report` is allowed only from `submitted`, transitions back to `in_preparation`, and stores the amendment reason in detail metadata. See `backend/app/annual_reports/services/status_service.py:265`.
- Financial summary totals income and recognized expenses; `taxable_income = total_income - recognized_expenses`. The detail response (`AnnualReportDetailResponse`) includes `total_income`, `total_expenses` (gross), `recognized_expenses`, `taxable_income`, `profit`, `tax_after_credits`, and `final_balance` (after subtracting paid advances). See `backend/app/annual_reports/services/financial_summary_service.py` and `backend/app/annual_reports/services/query_service.py`.
- Tax calculation is read-only. It uses financial summary, detail deductions/credits, credit-point rows/default resident points, income-tax engine, national-insurance engine, VAT net balance, and paid advances. `tax_after_credits` is the computed tax before advances; `final_balance = tax_after_credits - advances_paid` (negative = refund expected). See `backend/app/annual_reports/services/tax_service.py`.
- Persisting a tax calculation rejects requests with both `tax_due` and `refund_due`. See `backend/app/annual_reports/services/tax_service.py`.
- Income and expense line create/update/delete are financial mutations. A missing or soft-deleted related `ClientRecord` raises `CLIENT_RECORD.NOT_FOUND`; closed/frozen clients must not create, update, or delete annual-report income/expense lines. Successful manual income/expense mutations clear saved `tax_due` and `refund_due` while the report is still pre-submission (`not_started`, `collecting_docs`, `in_preparation`, or `pending_client`). See `backend/app/annual_reports/services/financial_line_helpers.py:55` and `backend/app/annual_reports/services/financial_line_service.py`.
- Income/expense source/category values are validated against enum values; audit entries are written for manual line mutations. See `backend/app/annual_reports/services/financial_line_service.py`.
- VAT auto-populate is a financial mutation and follows the same mutation rules as manual income/expense line changes. It requires a concrete actor id so audit cannot be silently skipped, raises `CLIENT_RECORD.NOT_FOUND` when the related `ClientRecord` is missing or soft-deleted, and is blocked for closed/frozen clients, including `force=True`; it writes audit records for created income lines, created expense lines, and deleted/replaced lines when `force=True`. Created-line audit payloads mark `source=vat_import`; force-replacement delete payloads mark `mutation_source=vat_import` and `mutation_reason=force_replace`. See `backend/app/annual_reports/services/vat_import_service.py:145`.
- VAT auto-populate is allowed only in `not_started`, `collecting_docs`, and `in_preparation`; existing income/expense lines cause `ANNUAL_REPORT.LINES_ALREADY_EXIST` unless `force=True`; import aggregates VAT by `client_record_id` and `tax_year`, maps VAT categories to annual expense categories, and writes lines only for positive generated totals. See `backend/app/annual_reports/services/vat_import_service.py:51`.
- VAT auto-populate response includes `skipped_items`, `warnings`, and `expense_breakdown`. `skipped_items` includes both true skipped totals and review items that need user attention. Zero generated expense category totals are returned as skipped items and do not create financial lines; zero income or no VAT data creates no line and no skipped noise. Negative generated totals are returned as skipped items with warnings and do not create financial lines automatically. A negative VAT source category inside a positive merged annual-report category is returned as `negative_source_contribution` with a warning while the positive net annual category can still create a line. `expense_breakdown` is the VAT source merge breakdown and includes both imported and skipped merged expense categories. Merged annual-report categories return `source_vat_categories`, and VAT-import expense audit payloads store the same source breakdown. See `backend/app/annual_reports/schemas/annual_report_financials.py:198` and `backend/app/annual_reports/services/vat_import_service.py:248`.
- Client freezing/closing integration cancels open annual reports by directly setting non-terminal reports to `canceled`. See `backend/app/annual_reports/repositories/report_repository.py:187`.

Future / planned:

- Dedicated financial-history endpoints for client annual-report comparisons are not implemented. See `docs/archive/annual-reports-legacy.md`.

## Error codes

Registry: `docs/architecture/error-codes.md`.

The annual reports namespace is registered as `ANNUAL_REPORT` in `docs/architecture/error-codes.md:43`. Codes raised by this module include:

- `ANNUAL_REPORT.NOT_FOUND`
- `ANNUAL_REPORT.INVALID_STATUS`
- `ANNUAL_REPORT.INVALID_TYPE`
- `ANNUAL_REPORT.INVALID_STAGE`
- `ANNUAL_REPORT.INVALID_STATUS_FOR_AMEND`
- `ANNUAL_REPORT.CLIENT_NOT_FOUND`
- `ANNUAL_REPORT.CONFLICT`
- `ANNUAL_REPORT.LINE_NOT_FOUND`
- `ANNUAL_REPORT.ANNEX_VALIDATION_ERROR`
- `ANNUAL_REPORT.TAX_CONFLICT`
- `ANNUAL_REPORT.INVALID_STATUS_FOR_AUTOPOPULATE`
- `ANNUAL_REPORT.LINES_ALREADY_EXIST`
- `ANNUAL_REPORT.AUDIT_ACTOR_REQUIRED`
- `ANNUAL_REPORT.SIGNER_NAME_MISSING`

Related code raised from this module but owned by another namespace:

- `CLIENT_RECORD.NOT_FOUND` when a related client record is missing or soft-deleted during annual-report signature setup or financial mutation validation.

Source grep: `backend/app/annual_reports/services/*`.

## Known issues

No open known issues.

## Resolved issues

- **F-001** (2026-06-04): Income/expense line update and delete paths checked only that the report existed, not that the line belonged to it. Fixed: repository mutation and audit snapshots are scoped by both `line_id` and `annual_report_id`.
- **F-002 / F-AR-001** (2026-06-04): Transitioning to `pending_client` silently skipped signature creation when client record or business could not be found. Fixed: transition now raises `CLIENT_RECORD.NOT_FOUND` or `ANNUAL_REPORT.SIGNER_NAME_MISSING` before the DB write. Source: `backend/app/annual_reports/services/status_signature_helper.py`.
- **F-003** (2026-06-04): Legacy docs described annual-report status transitions syncing linked tax-calendar entries and reminders. Retired as stale: `TaxCalendarEntry` has no status field; grouped calendar reads `AnnualReport.status` live; reminders domain has no coupling to annual-reports.
- **F-004** (2026-06-04): VAT auto-populate was flagged for aggregating by `client_record_id`+`tax_year` with no per-business selector. Accepted design: annual reports are client-scoped; Business is activity grouping only. Adding `business_id` filter would create a client-level report with a business-level import boundary — an inconsistency. See `backend/app/annual_reports/services/vat_import_service.py:201`.

## Decisions (preserved)

- The domain models full annual returns, not short refund-request form `0135`; primary form selection is controlled by `ClientAnnualFilingType -> PrimaryAnnualReportForm`. This is implemented in `FORM_MAP` and preserved from legacy summary context. See `backend/app/annual_reports/services/constants.py:8`.
- `6111` is modeled as an annex/schedule rather than a primary return. This is implemented as `form_6111` in `AnnualReportSchedule`. See `backend/app/annual_reports/models/annual_report_enums.py:45`.
- Annual reports are owned by client record and tax year, not by business. The root model, list queries, and VAT/advance/tax calculations use `client_record_id` and `tax_year`. See `backend/app/annual_reports/models/annual_report_model.py:43` and `backend/app/annual_reports/services/tax_service.py`.
- Annual-report financial line mutations fail closed when the related client record is missing or soft-deleted, and are locked for closed/frozen clients. This includes manual income/expense create, update, delete, and VAT auto-populate, including forced replacement. Manual income/expense mutations also clear persisted tax results before submission so readiness cannot rely on stale `tax_due` or `refund_due`.
- VAT auto-populate is a traceable financial mutation. It requires an actor id, audits created lines and deleted/replaced lines, marks the mutation source as VAT import, skips zero generated expense totals without creating lines, keeps empty zero income quiet, returns negative totals as skipped items/warnings, surfaces negative source contributions inside positive merged categories, and returns source VAT-category breakdown for merged annual-report categories.
- Annual reports carry a required `tax_calendar_entry_id` FK. Creation ensures an annual tax-calendar entry and links the report to it. This preserves the still-implemented tax-calendar decision from `backend/docs/domain_decisions_v3.md`. See `backend/app/annual_reports/models/annual_report_model.py:88` and `backend/app/annual_reports/services/create_service.py:109`.
- VAT auto-populate is intentionally client-wide. It aggregates all VAT work items for the report's `client_record_id`+`tax_year` across all businesses. Annual reports are client-scoped; Business is activity grouping only and is not the accounting boundary. Adding `business_id` filtering to auto_populate would create a client-level report with a business-level import boundary — an inconsistency. Do not add a `business_id` param. See `backend/app/annual_reports/services/vat_import_service.py:201` and `test_vat_auto_populate_aggregates_all_businesses_for_client_year`.
- Report-history list endpoints stay light: `GET /api/v1/clients/{client_record_id}/annual-reports` returns `AnnualReportListResponse`, and there is no current dedicated financial-history endpoint. This preserves the legacy product decision discussion while marking financial history as future/planned.
- Historical external government/legal sources are archived for reference only. They are not canonical implementation behavior and should not override code or architecture docs.

## Future / planned

- Add a dedicated client annual-report financial history endpoint, if the product wants a multi-year financial comparison table. Candidate paths discussed historically: `GET /api/v1/clients/{client_record_id}/annual-reports/history` or `GET /api/v1/annual-reports/{report_id}/client-history`. These do not exist in `backend/openapi.json` as of 2026-05-29.

## Historical notes

Archived legacy material: `docs/archive/annual-reports-legacy.md`.
