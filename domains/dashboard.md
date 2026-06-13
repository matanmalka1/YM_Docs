## Scope
This file owns only:
- Canonical current-state documentation for the dashboard domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Dashboard

The dashboard domain provides read-only aggregated widgets for the CRM home screen. It has no persistent tables of its own; instead it composes data from multiple other domains (clients, binders, charges, annual-reports, VAT reports, advance payments, work-queue, audit log) and returns derived response models. All endpoints are role-gated to `ADVISOR` and `SECRETARY`, with advisor-only sections returning empty payloads for secretaries rather than 403s.

Last verified against code + backend/openapi.json: 2026-06-10.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/dashboard/overview | Full management overview (role-differentiated payload) |
| GET | /api/v1/dashboard/tax-submissions | Annual-report submission statistics widget |

Both paths exist in `backend/openapi.json` (lines 4066 and 4010).

### Auth & roles

Both endpoints require `HTTPBearer` and `require_role(ADVISOR, SECRETARY)` (set at the router level). Advisor-only data (attention, quick_actions, advisor_today, recent_activity, open_charges) is silently omitted for secretaries — the endpoint returns 200 with empty/zero values, not 403.

Source: `backend/app/dashboard/api/dashboard_overview.py:8-12`, `dashboard_tax.py:8-12`.

### Query parameters

`GET /api/v1/dashboard/tax-submissions`
- `tax_year` (int, optional, `≥ 1900`): defaults to `date.today().year` when omitted.

Source: `backend/app/dashboard/api/dashboard_tax.py:19`, `backend/app/dashboard/services/dashboard_tax_service.py:23-26`.

## Model & fields

Dashboard has **no database tables**. All output is assembled from repositories of other domains and returned as Pydantic response models.

### DashboardOverviewResponse (`dashboard_extended.py:93`)

| Field | Type | Notes |
|-------|------|-------|
| `is_empty` | bool | True when `ClientRecordRepository.count() == 0` |
| `open_charges_count` | int | Advisor-only; 0 for secretary |
| `open_charges_amount_ils` | str \| None | ILS-formatted string e.g. `"₪1,234.5"`; null when count=0 or secretary |
| `vat_stats` | VatDashboardStats | Monthly + bimonthly VAT + advance-payment period stats |
| `attention` | AttentionResponse | Advisor-only; empty for secretary |
| `recent_activity` | list[RecentActivityItem] | Advisor-only; empty for secretary |

### TaxSubmissionWidgetResponse (`dashboard_tax.py:6`)

| Field | Type | Notes |
|-------|------|-------|
| `tax_year` | int | Requested or defaulted year |
| `total_clients` | int | Active business count |
| `reports_submitted` | int | SUBMITTED + CLOSED statuses |
| `reports_in_progress` | int | IN_PREPARATION + PENDING_CLIENT statuses |
| `reports_not_started` | int | total_clients − submitted − in_progress − material_collection |
| `submission_percentage` | float | `round(submitted/total_clients * 100, 1)` |
| `total_refund_due` | ApiDecimal | Sum from `AnnualReportRepository.sum_financials_by_year` |
| `total_tax_due` | ApiDecimal | Same query |

### AttentionBoardItem (`dashboard_extended.py:31`)

| Field | Type | Notes |
|-------|------|-------|
| `id` | str | Composite key `"{source_type}:{source_id}"`, regex `^\w+:\d+$` |
| `source_type` | str | WorkQueueSourceType value |
| `source_id` | int | ID in the originating domain |
| `title` | str | Work-queue item title |
| `client_name` | str \| None | Resolved via `load_client_profiles` |
| `due_date` | date \| None | |
| `days_delta` | int | `(due_date - today).days`; 0 if no due_date |
| `reason` | str \| None | Hebrew reason string per source type |
| `amount` | ApiDecimal \| None | Raw charge amount for charge items only; frontend formats as ILS |
| `urgency` | str | WorkQueueUrgency value |
| `href` | str | Deep-link path in the frontend |

### VatDashboardStats / VatDashboardPeriodStat (`dashboard_extended.py:62-80`)

Composed of `monthly`, `bimonthly`, and nested `advance_payments` (itself monthly + bimonthly). Each `VatDashboardPeriodStat` carries: `period`, `period_label`, `status_label`, `submitted`, `required`, `pending`, `completion_percent`.

### RecentActivityItem (`dashboard_extended.py:83`)

| Field | Type | Notes |
|-------|------|-------|
| `id` | int | Audit log row id (negative for binder lifecycle rows) |
| `date` | str | DD.MM.YYYY (Israel local time) |
| `time` | str | HH:MM (Israel local time) |
| `label` | str | Hebrew label from audit action/entity mapping |
| `client_name` | str | Resolved via bulk client read |
| `href` | str | Deep-link path |
| `activity_type` | str | One of `created`, `charge`, `done`, `updated` |

## Enums / statuses

Dashboard itself defines no enums. It references enums from other domains:

- `UserRole.ADVISOR`, `UserRole.SECRETARY` — `app/users/models/user.py`
- `WorkQueueUrgency` (`OVERDUE`, `APPROACHING`, `IMPORTANT`, `UPCOMING`) — work-queue domain; attention eligibility uses the first three (`_ATTENTION_URGENCIES`)
- `WorkQueueSourceType` (`VAT_WORK_ITEM`, `ANNUAL_REPORT`, `ADVANCE_PAYMENT`, `CHARGE`, `BINDER`, `TASK`) — work-queue domain
- `AnnualReportStatus` (`SUBMITTED`, `CLOSED`, `IN_PREPARATION`, `PENDING_CLIENT`, `COLLECTING_DOCS`) — annual-reports domain; used in tax-submission widget bucketing
- `BinderLocationStatus` (`READY_FOR_HANDOVER`, `HANDED_OVER`, `IN_OFFICE`) — binders domain; used to label binder lifecycle activity items

## Domain rules & invariants

**Overview role split** (`dashboard_overview_service.py`):
- `is_advisor = user_role == UserRole.ADVISOR`
- Attention, recent_activity, open_charges are advisor-only.
- Secretary receives: `is_empty`, zero `open_charges_*`, `vat_stats` (full), empty collections.
- No 403 is raised; differentiation is purely in data returned.

**Attention board selection** (`backend/app/dashboard/services/dashboard_attention_service.py:124-147`):
- Only built for `ADVISOR`; returns `[]` for all others.
- Fetches one small work-queue page for urgency tiers `{OVERDUE, APPROACHING, IMPORTANT}` (no client identity) then filters eligible items.
- `TASK` items require `due_date is not None` AND urgency in `{OVERDUE, APPROACHING, IMPORTANT}`.
- Sorted by urgency rank then `due_date`; capped at `_MAX_ITEMS = 7`.
- Client names resolved via bulk `load_client_profiles`.

**VAT stats period calculation** (`tax_status_stats_service.py:32-73`):
- Current monthly VAT period derived from `monthly_vat_period(reference_date)`.
- Current bimonthly VAT period derived from `bimonthly_vat_period(reference_date)`.
- Advance-payment bimonthly period derived from `bimonthly_advance_payment_period(reference_date)`.
- `status_label` is "מועד הגשה עבר" if `reference_date > due_date`, otherwise "ממתינה להגשה"; "הושלמה" when `pending == 0`; "אין דוחות נדרשים" when `required == 0`.
- Due dates materialised on demand via `TaxCalendarMaterializationService`.

**Tax-submission widget bucketing** (`dashboard_tax_service.py:30-45`):
- `submitted` = SUBMITTED + CLOSED
- `in_progress` = IN_PREPARATION + PENDING_CLIENT
- `material_collection` = COLLECTING_DOCS (counted internally; subtracted from not_started)
- `not_started` = `max(0, total_clients − submitted − in_progress − material_collection)` — clamped to prevent negative values when report counts exceed active business count.
- `total_clients` counts active businesses, not client records.

**Open charges** (`dashboard_overview_service.py:89-96`):
- Advisor-only; delegates to `ChargeRepository.open_charges_stats()`.
- Returns `(0, None)` for secretary or when count is 0.
- ILS formatted via `_format_ils` (strips trailing zeros, prepends `₪`).

**Recent activity** (`recent_activity_service.py:74-95`):
- Fetches up to 20 audit-log rows and 20 binder-lifecycle-log rows.
- Merges both streams, sorts descending by timestamp, returns top 5.
- Binder lifecycle IDs are negated (`-row.id`) to avoid collisions with audit log IDs.
- Timestamps localised to Israel timezone; tz-naive timestamps assumed UTC.
- Activity type for binder rows: `"done"` if `new_value == READY_FOR_HANDOVER`, else `"updated"`.

**Reference date** (`dashboard_overview_service.py:52-53`):
- Defaults to `israel_today()` when not supplied. Production call never passes it; test harness may inject a date.

## Error codes

No `DASHBOARD.*` error codes are raised in the live service layer.

Error envelope format: `docs/architecture/error-codes.md`.

## Known issues

No open known issues.

## Resolved issues

- **F-033** (2026-06-05): Stale service file references in README removed. `backend/app/dashboard/README.md` is now a pointer only.
- **F-034** (2026-06-05): `quick_actions` was a dead schema/service stub. Field removed.
- **F-035** (2026-06-05): `advisor_today` was a dead schema/service stub. Field and `AdvisorTodayService` removed.
- **F-036** (2026-06-05): `DASHBOARD.LIMIT_EXCEEDED` error code was documented but unreachable (referenced `DashboardExtendedService` which did not exist). Removed from docs.
- **F-037** (2026-06-05): `reports_not_started` can go negative when report counts exceed active-business count. Fixed with `max(0, ...)` clamp in `dashboard_tax_service.py`.
- **Phase 3** (2026-06-10): Attention-board calculation lives in `backend/app/dashboard/services/dashboard_attention_service.py`; dashboard exposes the result through `/api/v1/dashboard/overview`. Alerts remains a logical domain, not a backend package.

## Decisions (preserved)

From `backend/app/dashboard/README.md` (2026-03-17 audit):

- **No persistent tables.** Dashboard is a pure read-aggregation domain; all persistence belongs to source domains.
- **Role differentiation via empty payloads, not 403.** Both roles receive 200; secretary gets zeros/empty collections for advisor-only fields. Rationale: single endpoint, simpler frontend handling.
- **Attention board capped at 7 items** from the primary urgency tiers.

## Future / planned

None identified.

## Historical notes

No legacy `backend/docs` file covers the dashboard domain specifically. Two files contain tangential dashboard references:
- `backend/docs/frontend_screen_spec.md` — lists Dashboard as screen #2 in the navigation spec.
- `backend/docs/domain_decisions_v3.md` — references `due_date_effective` as the source of truth for overdue signals (applies to advance-payments dashboard widget via advance-payment domain, not dashboard itself).

No archive file is created for this domain (no legacy material to preserve).
