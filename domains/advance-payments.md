## Scope
This file owns only:
- Canonical current-state documentation for the advance-payments domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Advance Payments

The advance-payments domain manages periodic tax prepayments (מקדמות מס הכנסה) that Israeli legal entities submit to the Tax Authority on a monthly or bi-monthly basis. Each `AdvancePayment` record belongs to one `ClientRecord`, covers one reporting period (`YYYY-MM`), and tracks expected vs. paid amounts, payment status, and turnover snapshots used for amount calculation.

The expected amount formula is: `turnover_amount × advance_rate / 100 = calculated_amount`. An optional `override_amount` replaces `expected_amount` when set.

Last verified against code + backend/openapi.json: 2026-07-19.

## Endpoints

All paths confirmed present in `backend/openapi.json`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments` | List payments for a client (paginated; filter by year, status) |
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Read one payment owned by the client, including live-turnover enrichment when needed |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments` | Create a single advance payment (ADVISOR only) |
| `PATCH` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Update payment fields (ADVISOR only) |
| `DELETE` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Soft-delete a payment (ADVISOR only) |
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments/kpi` | Annual KPI aggregates for a client |
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments/prefill-turnover` | Look up VAT-report turnover for prefilling a new payment |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments/generate` | Generate full-year schedule for a client (ADVISOR only) |
| `GET` | `/api/v1/advance-payments/overview` | Cross-client overview (paginated; filter by year, month, status, exact `client_record_id`, legacy fuzzy `client_search`, etc.) |
| `GET` | `/api/v1/advance-payments/overview/batches` | Month-batch summaries for the overview grouping; supports exact `client_record_id` |
| `GET` | `/api/v1/annual-reports/{report_id}/advances-summary` | Advances summary scoped to an annual report (owned by annual_reports domain) |
| `GET` | `/api/v1/reports/advance-payments` | Reporting export (owned by reports domain) |

**Auth:** All advance-payments routes require `ADVISOR` or `SECRETARY` role; write operations (POST, PATCH, DELETE, generate) require `ADVISOR`.
Cite: `backend/app/advance_payments/api/advance_payments.py:23-27`, `advance_payments_overview.py:17-21`, `advance_payment_generate.py:12-14`.

## Model & fields

**Table:** `advance_payments`
Cite: `backend/app/advance_payments/models/advance_payment.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | No | autoincrement |
| `client_record_id` | int FK → `client_records.id` | No | operational anchor — never `legal_entity_id` |
| `period` | String(7) | No | `YYYY-MM` — first month of reporting period |
| `period_months_count` | int | No | `1` = monthly, `2` = bi-monthly |
| `due_date` | date | No | legacy compatibility field — usually 15th of month after period (see Future/planned) |
| `due_date_original` | date | Yes | immutable snapshot set on insert; once written, never changes |
| `due_date_effective` | date | Yes | effective due date for overdue/reminder logic; source of truth for all overdue checks |
| `due_date_override_reason` | String(500) | Yes | required when `due_date_effective ≠ due_date_original` |
| `expected_amount` | Numeric(10,2) | No | default 0.00; derived from `calculated_amount` or `override_amount` |
| `paid_amount` | Numeric(10,2) | No | default 0.00 |
| `turnover_amount` | Numeric(14,2) | Yes | snapshot of turnover at time of record; `NULL` = unknown, not zero |
| `advance_rate` | Numeric(5,2) | Yes | snapshot of advance rate at creation; frozen — changes to `LegalEntity.advance_rate` do not affect existing records |
| `calculated_amount` | Numeric(12,2) | No | `turnover_amount × advance_rate / 100`; derived display value |
| `override_amount` | Numeric(12,2) | Yes | replaces `expected_amount` when set |
| `status` | `AdvancePaymentStatus` enum | No | `pending \| paid \| partial` |
| `paid_at` | datetime | Yes | actual payment timestamp |
| `payment_method` | `PaymentMethod` enum | Yes | |
| `annual_report_id` | int FK → `annual_reports.id` | Yes | optional link to annual report |
| `tax_calendar_entry_id` | int FK → `tax_calendar_entries.id` (RESTRICT) | No | required — links to shared regulatory deadline |
| `notes` | String(500) | Yes | |
| `created_at` | datetime | No | set on insert |
| `updated_at` | datetime | Yes | set on update |
| `deleted_at` | datetime | Yes | soft-delete (via `SoftDeletableMixin`) |

**Indexes:**
- Partial unique index on `(client_record_id, period) WHERE deleted_at IS NULL` — prevents duplicate periods, allows soft-deleted re-creation.
- `idx_advance_payment_status`, `idx_advance_payment_due_date`, `idx_advance_payment_period_active`, `idx_advance_payment_calendar_entry_active`.

Cite: `backend/app/advance_payments/models/advance_payment.py:83-171`.

## Enums / statuses

**`AdvancePaymentStatus`** (`backend/app/advance_payments/models/advance_payment.py:58-63`):
| Value | Meaning |
|-------|---------|
| `pending` | Not yet paid |
| `paid` | Paid in full |
| `partial` | Partially paid |

`overdue` is **not** an enum value. It is a computed response field `timing_status` derived from `due_date` and `status` at read time.

**`PaymentMethod`** (`backend/app/advance_payments/models/advance_payment.py:66-74`):
| Value | Notes |
|-------|-------|
| `bank_transfer` | |
| `credit_card` | |
| `check` | |
| `direct_debit` | Most common for advance payments |
| `cash` | Rare; exists at post office bank |
| `other` | |

## Domain rules & invariants

Cite: `backend/app/advance_payments/services/advance_payment_service.py`.

- **Client status gate:** Creating a payment raises `ForbiddenError` if `ClientRecord.status` is `CLOSED` or `FROZEN`. (`advance_payment_service.py:48-53`)
- **Frequency validation:** `period_months_count` must match the client's `LegalEntity.advance_payment_frequency`. Mismatch raises `ADVANCE_PAYMENT.FREQUENCY_MISMATCH`. (`advance_payment_service.py:130-135`)
- **Bi-monthly start month:** Bi-monthly payments must start on an odd month (`1,3,5,7,9,11`). Violating raises `ADVANCE_PAYMENT.INVALID_PERIOD`. (`constants.py:23`, `advance_payment_service.py:68-71`)
- **No duplicate active period:** `UNIQUE(client_record_id, period) WHERE deleted_at IS NULL`. Duplicate insert raises `ADVANCE_PAYMENT.CONFLICT`. (`advance_payment_service.py:137-141`)
- **Frequency independence:** `advance_payment_frequency` must never be derived from `vat_reporting_frequency`. These are independent. (`domain_decisions_v3.md` §2, INV-07)
- **advance_rate snapshot frozen:** `advance_rate` is a snapshot at creation time. Changes to `LegalEntity.advance_rate` do not backfill existing records. (INV-06)
- **Amount calculation:** `calculated_amount = turnover_amount × advance_rate / 100` (ROUND_HALF_UP). `expected_amount = override_amount ?? calculated_amount`. (`advance_payment_service.py:74-92`)
- **Status is server-owned:** Clients cannot set `status` through the PATCH contract. The service derives it on create and whenever `paid_amount`, `expected_amount`, `turnover_amount`, or `override_amount` changes: `paid=0 → pending`, `paid ≥ expected → paid`, else `partial`.
- **Soft delete only:** Records are soft-deleted; hard deletes are not performed. (`advance_payment_service.py:244`)
- **Client-owned detail lookup:** Reading a single payment requires both `client_record_id` and `payment_id`. A missing, deleted, or differently owned payment returns `ADVANCE_PAYMENT.NOT_FOUND` instead of exposing another client's record. The lookup is independent of the list's active year, filters, and pagination.
- **Annual report invalidation hook:** When a payment is marked `paid`, the service invalidates any open annual report tax calculation for the same client+year. Failure is non-critical and does not fail the update. (`api/advance_payments.py:155-164`)
- **TaxCalendarEntry required:** Every payment must link to a `TaxCalendarEntry` (NOT NULL FK). The service calls `TaxCalendarMaterializationService.ensure_periodic_entry` to create or reuse the entry at creation time. (`advance_payment_service.py:152-157`)
- **due_date_original immutable:** Once set, `due_date_original` cannot change. Enforced by SQLAlchemy event listener. (`models/due_date_snapshot_events.py:24-31`)
- **due_date_effective requires reason:** If `due_date_effective ≠ due_date_original`, `due_date_override_reason` must be non-empty. Enforced on insert and update. (`models/due_date_snapshot_events.py:16-21`)
- **due_date_effective is overdue source of truth:** All overdue checks, badges, and reminders must use `due_date_effective`. Using `due_date_original` or `TaxCalendarEntry.due_date` is a bug. (INV-05)
- **anchor = client_record_id:** Workflow objects link to `ClientRecord`, never directly to `LegalEntity`. Joins to `LegalEntity` always go through `ClientRecord`. (INV per `domain_decisions_v3.md` §1)
- **Selected-client overview filters are exact:** The overview and overview batch endpoints accept `client_record_id` for exact `ClientRecord` matching. `client_search` remains a legacy fuzzy text filter for name, ID number, and office-client-number search.
- **Schedule generation:** `generate_annual_schedule` skips periods where `entry.due_date < reference_date` (default today) and skips periods that already have an active payment. (`advance_payment_service.py:269-287`)

**Computed response fields** (not stored; derived at serialization in `schemas/advance_payment.py`):
- `timing_status`: `"overdue"` if `status != paid AND today > due_date_effective`, else `"on_time"`. Falls back to `due_date` when `due_date_effective` is NULL (legacy rows).
- `paid_late`: `True` if `status == paid AND paid_at.date() > due_date_effective`. Falls back to `due_date` when `due_date_effective` is NULL.
- `delta`: `expected_amount - paid_amount`.
- `live_turnover`: populated by the router from `TurnoverLookupRepository` when `turnover_amount is None`.
- `missing_turnover`: `True` when `turnover_amount is None AND live_turnover is None`.
- `MonthBatchSummary.due_this_month_count`: count of non-paid payments whose effective due date falls in the current Israeli calendar month. The frontend must not infer this count from the reporting period month.

## Error codes

Codes follow `ADVANCE_PAYMENT.REASON` format. Registry: `docs/backend/error-codes.md`.

| Code | HTTP | Raised when |
|------|------|-------------|
| `ADVANCE_PAYMENT.CLIENT_RECORD_NOT_FOUND` | 404 | `client_record_id` does not exist |
| `ADVANCE_PAYMENT.FREQUENCY_NOT_SET` | 404 | Client has no configured `advance_payment_frequency` |
| `ADVANCE_PAYMENT.INVALID_PERIOD` | 409 | `period_months_count` unsupported, or bi-monthly period starts on even month |
| `ADVANCE_PAYMENT.FREQUENCY_MISMATCH` | 409 | Request `period_months_count` does not match client's configured frequency |
| `ADVANCE_PAYMENT.CONFLICT` | 409 | Active payment already exists for `(client_record_id, period)` |
| `ADVANCE_PAYMENT.NOT_FOUND` | 404 | Payment ID not found for the given client |
| `ADVANCE_PAYMENT.RATE_INVALID` | 400 | VAT rate is zero when attempting reverse calculation (`advance_payment_calculator.py`) |
| `CLIENT.CLOSED` | 403 | Client is closed — cannot create payment |
| `CLIENT.FROZEN` | 403 | Client is frozen — cannot create payment |

## Known issues

No open known issues.

## Resolved issues

- **F-005** (2026-06-05): `timing_status` and `paid_late` were computed from legacy `due_date`, ignoring `due_date_effective`. Fixed: response schemas now use `due_date_effective or due_date` for both derived values (`backend/app/advance_payments/schemas/advance_payment.py:50-62,153-155`).

## Decisions (preserved)

From `backend/docs/domain_decisions_v3.md` (v3.1, May 2026) and the archived legacy spec at `docs/archive/advance-payments-legacy.md`:

1. **`overdue` is computed, not stored.** Removed from the status enum. `timing_status` (`overdue | on_time`) is derived at read time from `due_date_effective or due_date` and `status`. `paid_late` is similarly computed from `paid_at` versus the effective due date. Decision confirmed in advance_payments_spec.md §Closed Decisions and current schemas.

2. **Turnover snapshot vs. live.** `turnover_amount` is a snapshot frozen on write. For pending/partial payments, `live_turnover` is fetched from `VatWorkItem` at read time via `TurnoverLookupRepository` when `turnover_amount is None`. No hard dependency on VAT report existing before advance payment.

3. **Edit via drawer, not inline.** UI uses a drawer component for editing payments (UI decision, not enforced in backend).

4. **Overview grouped by month (collapsed by default).** Month-batch summaries are provided by `/overview/batches`. (UI decision.)

5. **`advance_rate` default from `LegalEntity`.** At creation time, if `advance_rate` is not passed, the service reads it from `LegalEntity.advance_rate`. The value is then snapshotted frozen on the record.

6. **`due_date_original` immutable; `due_date_effective` is overdue source of truth.** Architectural invariants INV-04 and INV-05. Enforced by SQLAlchemy event listeners.

7. **Workflow objects anchor on `client_record_id`, not `legal_entity_id`.** Invariant from `domain_decisions_v3.md` §1.

8. **Frequency independence.** `advance_payment_frequency` never derived from `vat_reporting_frequency`. Enforced in service and documented as INV-07.

9. **`TaxDeadline` removed.** All deadline lookups go through `TaxCalendarEntry`. New code must not use the `TaxDeadline` name or concept. (`domain_decisions_v3.md` §3.6)

## Future / planned

These items are explicitly **not yet implemented**. Do not describe as current behavior.

- **Remove legacy `AdvancePayment.due_date`.** The old `due_date` column coexists with `due_date_original` and `due_date_effective`. Plan: audit all consumers to use `due_date_effective`, then drop `due_date`. (`domain_decisions_v3.md` §3.3, §9)
- **Explicit due-date override endpoint.** No dedicated endpoint for updating `due_date_effective` exists yet. If added, it must enforce `due_date_override_reason`, permission checks, and must block updates to terminal-state records. (`domain_decisions_v3.md` §9, INV-09)
- **`reported_turnover` / `turnover_source_vat_report_id` snapshot fields.** The legacy spec proposed storing a VAT-report snapshot ID at payment time (`advance_payments_spec.md` §שינויים נדרשים). These fields do **not exist** in the current model; the current approach uses `turnover_amount` as an unlinked snapshot and `TurnoverLookupRepository` for live lookups.
- **Turnover drift warning.** Legacy spec proposed a ⚠ alert when the VAT report's turnover changed after the advance payment was recorded. Not implemented.
- **`missing_turnover` blocking batch "mark ready".** Legacy spec proposed that `missing_turnover` blocks batch operations. Currently `missing_turnover` is a read signal only and does not block individual or batch writes.
- **Optional rename of `advance_rate` to `rate_used`.** Considered in `domain_decisions_v3.md` §3.3; deferred — current code uses `advance_rate`.

## Historical notes

Legacy spec archived at `docs/archive/advance-payments-legacy.md`.

Historical source archived at `docs/archive/advance-payments-legacy.md`.
