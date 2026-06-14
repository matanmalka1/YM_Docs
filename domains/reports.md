## Scope
This file owns only:
- Canonical current-state documentation for the reports domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Reports

The reports domain provides read-only management and operational reports that aggregate data from other domains (charge, vat-reports, advance-payments, annual-reports, clients). It defines no persistent tables of its own; all output is computed at request time and returned as derived payloads. Export endpoints additionally serialize reports to Excel or PDF files written to a server-side temp directory.

Last verified against code + backend/openapi.json: 2026-06-14 for aging export binary OpenAPI docs; full-domain verification remains 2026-05-29.

## Endpoints

All routes require role `ADVISOR` (`require_role(UserRole.ADVISOR)` router-level dependency, `backend/app/reports/api/reports.py:25`). Authenticated via `HTTPBearer`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/reports/vat-compliance | VAT compliance summary by year |
| GET | /api/v1/reports/advance-payments | Advance-payment collections summary by year / month |
| GET | /api/v1/reports/annual-reports | Annual-report status breakdown by tax year |
| GET | /api/v1/reports/aging | Accounts-receivable aging report (as-of date) |
| GET | /api/v1/reports/aging/export | Download aging report as Excel or PDF |

(All paths confirmed in `backend/openapi.json`.)

### Query parameters

| Endpoint | Parameter | Type | Required |
|----------|-----------|------|----------|
| /vat-compliance | `year` | int | yes |
| /advance-payments | `year` | int | yes |
| /advance-payments | `month` | int | no |
| /annual-reports | `tax_year` | int | yes |
| /aging | `as_of_date` | date | no (defaults to today) |
| /aging/export | `format` | `excel` \| `pdf` | yes |
| /aging/export | `as_of_date` | date | no (defaults to today) |

## Model & fields

This domain has **no persistent database models**. All response models are in `backend/app/reports/schemas.py`.

### VatComplianceReportResponse (`schemas.py:33`)
Top-level: `year`, `total_clients`, `items: list[VatComplianceReportItemResponse]`, `stale_pending: list[VatComplianceStalePendingResponse]`.

**VatComplianceReportItemResponse** (`schemas.py:11`):
| Field | Type | Notes |
|-------|------|-------|
| client_record_id | int | FK → clients domain |
| client_name | str | Resolved from LegalEntity.official_name |
| year | int | |
| period_type | str | Enum value from VatWorkItem.period_type |
| grouping_key | str | Synthetic: `"{client_record_id}:{year}:{period_type}"` |
| periods_expected | int | |
| periods_filed | int | |
| periods_open | int | Computed: periods_expected − periods_filed |
| on_time_count | int | Filed items where filed_at.date() ≤ due_date_effective |
| late_count | int | Filed items where filed_at.date() > due_date_effective |
| compliance_rate | ApiDecimal | Percentage: periods_filed / periods_expected × 100 |

**VatComplianceStalePendingResponse** (`schemas.py:26`): `client_record_id`, `client_name`, `period`, `days_pending`. Stale threshold: `VAT_STALE_PENDING_DAYS = 30` (`constants.py:7`).

### AdvancePaymentCollectionsReportResponse (`schemas.py:50`)
`year`, `month | None`, `total_expected`, `total_paid`, `collection_rate`, `total_gap`, `items: list[AdvancePaymentReportItemResponse]`.

**AdvancePaymentReportItemResponse** (`schemas.py:40`):
| Field | Type | Notes |
|-------|------|-------|
| client_record_id | int | |
| office_client_number | int \| None | From ClientRecord |
| client_name | str | Resolved from LegalEntity.official_name; falls back to `"לקוח #{id}"` if legal entity missing |
| total_expected | ApiDecimal | |
| total_paid | ApiDecimal | |
| overdue_count | int | |
| gap | ApiDecimal | total_expected − total_paid |

### AnnualReportStatusReportResponse (`schemas.py:74`)
`tax_year`, `total`, `statuses: list[AnnualReportStatusGroupResponse]`.

**AnnualReportStatusGroupResponse** (`schemas.py:68`): `status: AnnualReportStatus`, `count`, `clients: list[AnnualReportStatusClientResponse]`. Only statuses with at least one client are included in the response.

**AnnualReportStatusClientResponse** (`schemas.py:60`): `client_record_id`, `client_name`, `form_type: PrimaryAnnualReportForm | None`, `filing_deadline: ApiDateTime | None`, `days_until_deadline: int | None`.

### AgingReportResponse (`schemas.py:100`)
`report_date`, `total_outstanding`, `items: list[AgingReportItemResponse]`, `summary: AgingReportSummaryResponse`, `total`, `page`, `page_size`.

**AgingReportItemResponse** (`schemas.py:80`):
| Field | Type | Notes |
|-------|------|-------|
| client_record_id | int | |
| client_name | str | Resolved from LegalEntity.official_name |
| total_outstanding | float | |
| current | float | 0–30 days outstanding |
| days_30 | float | 31–60 days |
| days_60 | float | 61–90 days |
| days_90_plus | float | 91+ days |
| oldest_invoice_date | date \| None | Date of oldest unpaid issued charge |
| oldest_invoice_days | int \| None | Days since oldest invoice |

**AgingReportSummaryResponse** (`schemas.py:92`): `total_clients`, `total_current`, `total_30_days`, `total_60_days`, `total_90_plus`.

## Enums / statuses

This domain imports enums from other domains; it defines none of its own.

| Enum | Source | Used in |
|------|--------|---------|
| `AnnualReportStatus` | `backend/app/annual_reports/models/annual_report_enums.py` | AnnualReportStatusGroupResponse.status |
| `PrimaryAnnualReportForm` | `backend/app/annual_reports/models/annual_report_enums.py` | AnnualReportStatusClientResponse.form_type |

VAT period type values come from `VatWorkItem.period_type` (vat-reports domain).

## Domain rules & invariants

**Aging report** (`backend/app/reports/services/reports_service.py`):
- Only considers charges with status `issued` and non-null `issued_at` (enforced in `ChargeRepository.get_aging_buckets_paginated` / `get_aging_totals`).
- `as_of_date` defaults to `date.today()` when not provided (`reports_service.py:32`).
- Results are ordered by `total_outstanding` descending before pagination.
- `GET /api/v1/reports/aging` supports `page` and `page_size`; totals and summary fields reflect the full filtered report, not just the current page.

**VAT compliance report** (`backend/app/reports/services/vat_compliance_report.py`):
- On-time vs. late counts compare `filed_at.date()` against `due_date_effective` (the effective deadline, not legacy `due_date`).
- Stale-pending items are fetched for the full year and filtered in Python: only items with `days_pending ≥ VAT_STALE_PENDING_DAYS` (30) are included in the response (`vat_compliance_report.py:77`).
- Clients missing from the name map are silently skipped from `items` (`vat_compliance_report.py:40–41`).

**Advance-payment collections report** (`backend/app/reports/services/advance_payment_report.py`):
- `collection_rate` = `total_paid / total_expected × 100`; returns 0.0 if `total_expected` is zero (`advance_payment_report.py:51`).
- Client name falls back to `"לקוח #{id}"` when the legal entity cannot be resolved (`advance_payment_report.py:38–40`).

**Annual-report status report** (`backend/app/reports/services/annual_report_status_report.py`):
- Groups all annual reports for the given tax year by their `AnnualReportStatus`.
- Statuses with zero clients are omitted from the response (`annual_report_status_report.py:44–50`).
- `days_until_deadline` is negative when the deadline has already passed.

**Aging export** (`backend/app/reports/services/reports_export_service.py`):
- Format validated at router level with regex `^(excel|pdf)$` (`reports.py:69`).
- Excel export requires `openpyxl`; PDF export requires `reportlab`. Missing libraries raise HTTP 500 with explicit install guidance (`reports_export_service.py:38–43`).
- Files are written to `EXPORT_TEMP_DIR` (`/tmp/.../exports`); the endpoint returns a `FileResponse` with `Content-Disposition: attachment` (`reports.py:81`).
- OpenAPI documents both possible successful download media types, Excel and PDF, as binary file responses rather than `application/json` with an empty schema (`backend/app/reports/api/reports.py:74-78`, `backend/openapi.json`).

## Error codes

This domain does not define `REPORTS.REASON` error codes. The export endpoint raises raw `HTTPException` (HTTP 500) for missing export libraries and unexpected export failures (`reports_export_service.py:38,43`). All other errors propagate from dependency domains via their own error envelopes.

See `docs/backend/error-codes.md` for the global error envelope format.

## Known issues

No open known issues.

## Resolved issues

- **F-038** (2026-06-05): `reporting_frequency` was a duplicate of `period_type` in `VatComplianceReportItemResponse`. Field removed from `schemas.py` and `vat_compliance_report.py`. Consumers use `period_type`.

## Decisions (preserved)

No standalone domain decisions for `reports` were found in `backend/docs/domain_decisions_v3.md`. The module was designed as a pure read/aggregation layer with no domain-owned persistent tables. Cross-domain aggregation over charge, vat-reports, advance-payments, annual-reports, and clients is intentional.

## Future / planned

None identified in code or legacy docs.

## Historical notes

No legacy files specific to this domain exist under `backend/docs/`. The `backend/app/reports/README.md` was an up-to-date module-level reference; this canonical doc supersedes it for cross-domain consumers.
