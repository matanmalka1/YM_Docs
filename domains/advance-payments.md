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

Last verified against code + backend/openapi.json: 2026-07-26.

## Endpoints

All paths confirmed present in `backend/openapi.json`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments` | List payments for a client (paginated; filter by year, status) |
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Read one payment owned by the client, including `available_turnover` enrichment when the period is not snapshotted |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments` | Create a single advance payment (ADVISOR only) |
| `PATCH` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Update payment fields (ADVISOR only) |
| `DELETE` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}` | Soft-delete a payment (ADVISOR only) |
| `GET` | `/api/v1/clients/{client_record_id}/advance-payments/kpi` | Annual KPI aggregates for a client |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments/{payment_id}/refresh-turnover` | Snapshot the period's VAT turnover onto the payment and recompute amounts (ADVISOR only) |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments/refresh-turnover` | Bulk snapshot for explicitly listed `payment_ids`; filed returns only, non-atomic (ADVISOR only) |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments/generate` | Generate full-year schedule for a client (ADVISOR only) |
| `POST` | `/api/v1/clients/{client_record_id}/advance-payments/bulk-rate-update` | Reprice the client's pending periods from a month onward and rewrite the legal-entity default rate (ADVISOR only; idempotency key required) |
| `GET` | `/api/v1/advance-payments/overview` | Cross-client overview (paginated; filter by year, month, status, `timing_status`, `vat_mismatch`, exact `client_record_id`, legacy fuzzy `client_search`, etc.) |
| `GET` | `/api/v1/advance-payments/overview/batches` | Month-batch summaries for the overview grouping; supports exact `client_record_id` |
| `POST` | `/api/v1/advance-payments/bulk-mark-paid` | Top up explicitly listed payments to their expected amount; unpayable rows are reported as skips (ADVISOR only; idempotency key required) |
| `GET` | `/api/v1/advance-payments/bulk-generate/preview` | Eligible-client count plus active clients that have no frequency configured. Takes no `year` — eligibility is a client property, not a year's |
| `POST` | `/api/v1/advance-payments/bulk-generate` | Generate annual schedules for one server-sized chunk of eligible clients; repeat with the returned `next_cursor` until it is `null` (ADVISOR only; one idempotency key per chunk) |
| `GET` | `/api/v1/annual-reports/{report_id}/advances-summary` | Advances summary scoped to an annual report (owned by annual_reports domain) |
| `GET` | `/api/v1/reports/advance-payments` | Reporting export (owned by reports domain) |

**Auth:** All advance-payments routes require `ADVISOR` or `SECRETARY` role; write operations (POST, PATCH, DELETE, generate) require `ADVISOR`.
Cite: `backend/app/advance_payments/api/advance_payment_routes.py`, `advance_payment_routes_overview.py`, `advance_payment_routes_generate.py`.

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
| `turnover_source` | `TurnoverSource` enum | Yes | provenance of `turnover_amount`; `NULL` exactly when `turnover_amount` is `NULL` |
| `turnover_snapshot_at` | datetime | Yes | when `turnover_amount` was frozen; also `NULL` on rows backfilled by migration `8a1c47d0b3e2` |
| `advance_rate` | Numeric(5,2) | Yes | snapshot of advance rate at creation; frozen — changes to `LegalEntity.advance_rate` do not affect existing records |
| `calculated_amount` | Numeric(12,2) | No | `turnover_amount × advance_rate / 100`; derived display value |
| `override_amount` | Numeric(12,2) | Yes | replaces `expected_amount` when set; wins even over `withheld_amount` |
| `withheld_amount` | Numeric(12,2) | Yes | withheld-at-source credit (ניכוי במקור); subtracted from `calculated_amount` when deriving `expected_amount`. `NULL` = none entered, treated as zero |
| `status` | `AdvancePaymentStatus` enum | No | `pending \| paid \| partial` |
| `paid_at` | datetime | Yes | actual payment timestamp |
| `payment_method` | `PaymentMethod` enum | Yes | |
| `payment_reference` | String(100) | Yes | bank/authority reference (אסמכתה) of the payment, as reported by the client |
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

**`TurnoverSource`** (`backend/app/advance_payments/models/advance_payment.py`):
| Value | Meaning |
|-------|---------|
| `manual` | Typed by an advisor (create or PATCH) |
| `vat_filed` | Snapshotted by the refresh command; every covered month was filed |
| `vat_pending` | Snapshotted by the refresh command with `confirm_pending`; at least one covered month was not filed |

**`PaymentMethod`** (`backend/app/advance_payments/models/advance_payment.py:66-74`):
| Value | Notes |
|-------|-------|
| `bank_transfer` | |
| `credit_card` | |
| `check` | |
| `direct_debit` | Most common for advance payments |
| `cash` | Rare; exists at post office bank |
| `other` | |


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
  submitted obligation takes an amendment, which arrives in a later wave.
- **Cancel from any unlocked stage**, and `canceled` is terminal.

`RESOLVED_OBLIGATION_STATUSES` = `{submitted, canceled}` is the single answer to
"does this need further work?", read by both the Python predicate and every SQL
query. `OBLIGATION_STATUS_LABELS` is the single Hebrew vocabulary.

## Domain rules & invariants

**Which periods are owed.** `app/common/obligation_plan.py` is the single answer. It is narrowed by the client's configured frequency **and** by that obligation type's liability range on `LegalEntity` — per type, because the types move independently. A period is owed when it *intersects* the range, so a period the client was liable for on any of its days is created in full. NULL on either side is unbounded. See `docs/domains/clients.md` § Liability ranges.

Cite: `backend/app/advance_payments/services/advance_payment_service.py`.

- **Client status gate:** Creating a payment goes through the shared client-eligibility guard `assert_client_record_is_active` (`backend/app/clients/guards/client_record_guards.py`), which raises `ConflictError` / `CLIENT_RECORD.CLOSED` (409) for any non-`ACTIVE` client. The guard is an allowlist, so a status added later fails closed. Its SQL twin, `eligible_client_status_expr` (`backend/app/clients/repositories/client_active_scope.py`), scopes office-wide generation and must change together with it. This domain previously re-derived the rule locally and raised 403 `CLIENT.CLOSED` / `CLIENT.FROZEN`.
- **Frequency validation:** `period_months_count` must match the client's `LegalEntity.advance_payment_frequency`. Mismatch raises `ADVANCE_PAYMENT.FREQUENCY_MISMATCH`. (`advance_payment_service.py:130-135`)
- **Bi-monthly start month and supported frequency:** neither rule is this domain's. Both are enforced once by `TaxCalendarMaterializationService` during materialization — `_validate_period_alignment` raises `TAX_CALENDAR.INVALID_PERIOD_ALIGNMENT` for an even bi-monthly start month, and `_periodic_rule_type` raises `TAX_CALENDAR.INVALID_PERIOD_FREQUENCY` for a `period_months_count` outside `{1, 2}`. This domain's local copies and its `ADVANCE_PAYMENT.INVALID_PERIOD` code were retired, along with the duplicate Pydantic validator on `AdvancePaymentCreateRequest` that shadowed them with a 422. See `docs/domains/tax-calendar.md`.
- **No duplicate active period:** `UNIQUE(client_record_id, period) WHERE deleted_at IS NULL`. Duplicate insert raises `ADVANCE_PAYMENT.CONFLICT`. (`advance_payment_service.py:137-141`)
- **Frequency independence:** `advance_payment_frequency` must never be derived from `vat_reporting_frequency`. These are independent. (`domain_decisions_v3.md` §2, INV-07)
- **advance_rate snapshot frozen:** `advance_rate` is a snapshot at creation time. Changes to `LegalEntity.advance_rate` do not backfill existing records. (INV-06)
- **turnover snapshot frozen:** `turnover_amount` never follows later amendments to the VAT return it came from. Filing or amending a `VatWorkItem` writes nothing to `AdvancePayment`; the only paths that set `turnover_amount` are create, PATCH, and the refresh command.
- **turnover_source tracks the writer:** create and PATCH set `manual`; the refresh command sets `vat_filed` or `vat_pending`. A PATCH that overwrites a VAT-sourced turnover resets the source to `manual`, so the label never outlives the figure it described. Clearing `turnover_amount` clears both provenance columns.
- **Refresh is all-or-nothing per period:** the command snapshots only when *every* month covered by `period_months_count` has a VAT work item, so a half-covered bi-monthly period cannot snapshot a silently halved turnover. The result is `vat_filed` only when every covered month is filed; a mixed filed/unfiled bi-monthly period is `vat_pending`.
- **Unfiled VAT requires explicit confirmation:** refreshing from a `READY_FOR_REVIEW` return raises `ADVANCE_PAYMENT.VAT_NOT_FILED` unless the request passes `confirm_pending`. The chosen source is recorded on the row and in the audit entry.
- **Refresh does not override an override:** the command recomputes `calculated_amount`, but `expected_amount` still resolves to `override_amount` when one is set.
- **Bulk refresh takes explicit ids, never a filter:** `payment_ids` is required (1..`MAX_BULK_REFRESH_PAYMENTS`). A filter-based form is deliberately absent — a filter can match rows the advisor never saw, and this command writes to every row it matches. Ownership of every id is checked before any write, so a foreign or unknown id fails the whole request with `ADVANCE_PAYMENT.NOT_FOUND`.
- **Bulk refresh never snapshots unfiled returns:** there is no bulk `confirm_pending`. Confirming an unfiled figure is a judgement about one period and cannot be given meaningfully for a batch, so `vat_pending` periods are counted in `skipped_not_filed` instead.
- **Bulk refresh never touches a settled payment:** rows with `status = paid` are counted in `skipped_paid`. Snapshotting recomputes `expected_amount` and can drop a PAID row to PARTIAL; recording a payment before its VAT return arrives is normal, so PAID rows with `turnover_amount IS NULL` are common. The single-payment command still allows it — that is the deliberate escape hatch for correcting one settled row.
- **Bulk refresh is not atomic:** each period is an independent business fact, so a period that cannot be snapshotted is counted, not raised, and does not roll back its neighbours. Skips are split into `skipped_no_vat` and `skipped_not_filed` because the two demand different follow-ups (chase the return vs. wait for filing). Every refreshed payment gets its own `advance_payment.turnover_refreshed` audit entry — there is no grouped batch entry.
- **One turnover rule, one implementation:** `TurnoverLookupRepository._resolve` is the only place that decides what a period can draw from VAT. `resolve_turnover` (one period), `resolve_turnover_for_client`, and `resolve_turnover_for_clients` differ only in how many periods they ask about. Never add a fourth path or re-derive the coverage/source rule at a call site.
- **The mismatch filter is that same rule in SQL:** `vat_turnover_mismatch_expr` (same module) is the only SQL form of the coverage-plus-tolerance rule, and exists because a filter narrows a set the server never loads — the overview is paginated, so a Python check could not back it. It and `_resolve`/`VatTurnoverMismatch.from_comparison` must change together: a row the `vat_mismatch` filter keeps must be a row that carries the flag. It is used by the overview filter (both directions: `true` keeps mismatching rows, `false` keeps the rest) and by `MonthBatchSummary.vat_mismatch_count`. Bi-monthly months are compared within the period's own year, which is safe because a bi-monthly period starts on an odd month and therefore never crosses a year end.
- **Amount calculation:** `calculated_amount = turnover_amount × advance_rate / 100` (ROUND_HALF_UP), and stays **gross** — `withheld_amount` is never folded into it. `expected_amount` resolves in order: `override_amount` when set (it is the final say, and wins over `withheld_amount`); otherwise `max(0, calculated_amount - withheld_amount)`, floored at zero. (`advance_payment_service.py:196-216`)
- **Status is server-owned:** Clients cannot set `status` through the PATCH contract. The service derives it on create and whenever `paid_amount`, `expected_amount`, `turnover_amount`, or `override_amount` changes: `paid=0 → pending`, `paid ≥ expected → paid`, else `partial`.
- **Soft delete only:** Records are soft-deleted; hard deletes are not performed. (`advance_payment_service.py:244`)
- **Client-owned detail lookup:** Reading a single payment requires both `client_record_id` and `payment_id`. A missing, deleted, or differently owned payment returns `ADVANCE_PAYMENT.NOT_FOUND` instead of exposing another client's record. The lookup is independent of the list's active year, filters, and pagination.
- **Detail period navigation is client/year scoped:** The detail screen's picker and previous/next controls list active payments for the current payment's `client_record_id` and calendar year only. Siblings are ordered chronologically by `period`; navigation preserves whether the user entered through the organization list or the client tab. The controls are disabled while the edit form has unsaved changes.
- **Annual report invalidation hook:** When a payment is marked `paid`, the service invalidates any open annual report tax calculation for the same client+year. Failure is non-critical and does not fail the update. (`api/advance_payments.py:155-164`)
- **TaxCalendarEntry required:** Every payment must link to a `TaxCalendarEntry` (NOT NULL FK). The service calls `TaxCalendarMaterializationService.ensure_periodic_entry` to create or reuse the entry at creation time. (`advance_payment_service.py:152-157`)
- **due_date_original immutable:** Once set, `due_date_original` cannot change. Enforced by SQLAlchemy event listener. (`models/due_date_snapshot_events.py:24-31`)
- **due_date_effective requires reason:** If `due_date_effective ≠ due_date_original`, `due_date_override_reason` must be non-empty. Enforced on insert and update. (`models/due_date_snapshot_events.py:16-21`)
- **due_date_effective is overdue source of truth:** All overdue checks, badges, and reminders must use `due_date_effective`. Using `due_date_original` or `TaxCalendarEntry.due_date` is a bug. (INV-05)
- **anchor = client_record_id:** Workflow objects link to `ClientRecord`, never directly to `LegalEntity`. Joins to `LegalEntity` always go through `ClientRecord`. (INV per `domain_decisions_v3.md` §1)
- **Selected-client overview filters are exact:** The overview and overview batch endpoints accept `client_record_id` for exact `ClientRecord` matching. `client_search` remains a legacy fuzzy text filter for name, ID number, and office-client-number search.
- **Schedule generation:** `generate_annual_schedule` skips periods where `entry.due_date < reference_date` (default today) and skips periods that already have an active payment. (`advance_payment_service.py:269-287`)
- **Office-wide generation is chunked, and the server owns the chunking:** `bulk_generate_annual_schedules` walks eligible clients by keyset on `ClientRecord.id` and returns `next_cursor`; the caller repeats until it is `null`. Batch size (`BULK_GENERATE_CLIENT_CHUNK_SIZE`) and ordering are server decisions — a caller-chosen batch could silently omit clients. Keyset, not offset: a run spans several requests, and an offset would skip or repeat clients if the eligible set changed underneath it. Each chunk needs its own idempotency key so retrying one chunk cannot double-create.
- **Office-wide eligibility is `ACTIVE` + a configured frequency:** closed and frozen clients are excluded by the query, not skipped later — creating an advance for them is already forbidden. Active clients with no `advance_payment_frequency` are excluded from generation but *reported* by the preview endpoint: unlike a closed client or an already-generated period, a missing frequency leaves the client with no schedule at all and is a data gap the advisor has to close.
- **Office-wide generation is not atomic across clients:** a client whose own generation raises an `AppError` is collected into `failed` and the chunk continues — one misconfigured client must not cost the chunk its other clients. A database error still fails the whole chunk, because the transaction is no longer trustworthy; the caller retries that chunk under the same idempotency key.
- **A cadence change blocks generation until it is confirmed:** `exists_for_period` matches on the `YYYY-MM` key alone, so rows created under the client's previous `advance_payment_frequency` occupy the periods the new schedule needs. When such rows exist and `cleanup_stale_cadence` is not set, the generator creates **nothing** — generating only the periods the stale rows do not occupy is what leaves a month covered by both cadences at once. The response reports the counts and the caller repeats with the flag. A client whose only stale rows are settled is not blocked, since no cleanup could free those periods.
- **The cleanup sweep is future-only and PENDING-only:** it covers rows in the generated year whose `period_months_count` differs from the configured one and whose effective due date is still ahead. Past-due unpaid rows are never removed — an overdue period is a debt, not a leftover. Paid and part-paid rows are never removed either, and are reported as `stale_cadence.settled`: that period keeps the old shape until someone resolves it by hand. Deletions are soft and audited as `advance_payment.deleted` with a system-written `reason`.
- **Bulk-created schedules are marked in the audit trail:** rows created by the office-wide run carry `source = "bulk_generate"` in the `advance_payment.created` audit metadata, reusing the existing action rather than adding a new one.

**Closing facts** (stored, written once when an advisor closes the period — W3, D-13/D-15/D-20/D-32):
- `closed_at` / `closed_by`: when and by whom the period was closed (status → `submitted`). `paid_at` stays the *payment* event; the two differ when an advisor settles a part-paid or unpaid period (D-16).
- `closed_late`: the Israel-local date of `closed_at` compared to `due_date_effective or due_date`, recorded at the close. NULL is reserved for "no due date at close" (D-32) — advances always have a due date today, so it is a real boolean until amendments (W4) arrive.
- `assigned_to`: nullable FK to `users.id` — required by the closing gate (D-15), editable via PATCH until the close.
- The close is advisor-only via `POST /clients/{client_record_id}/advance-payments/{payment_id}/status` (single-step transition through the shared graph: forward one stage, back one with a required note, cancel, or close). Closing asserts the shared readiness gate first. `GET .../{payment_id}/readiness` publishes the same gate as `{is_ready, issues}` — assignee set, turnover known, expected amount computable. **Payment in full is not a gate** (D-16).
- A submitted advance is fully locked (D-13): PATCH, delete, turnover refresh, and transitions out are rejected with `OBLIGATION.LOCKED`; bulk mark-paid and bulk refresh skip closed rows (`reason: "closed"` / `skipped_closed`).

**Computed response fields** (not stored; derived at serialization in `schemas/advance_payment.py`):
- `timing_status`: `"overdue"` if `status != submitted AND today > due_date_effective`, else `"on_time"`. Falls back to `due_date` when `due_date_effective` is NULL (legacy rows).
- `delta`: `expected_amount - paid_amount`.
- `available_turnover`: `{amount, source}` or `null`. Populated by the router from `TurnoverLookupRepository` only when `turnover_amount is None`. It is what the period *could* be snapshotted from — never the payment's turnover — and feeds no amount on the record. `source` is `vat_filed | vat_pending`; `manual` cannot appear. Clients must not render it in the same slot as `turnover_amount`.
- `missing_turnover`: `True` when `turnover_amount is None AND available_turnover is None`.
- `MonthBatchSummary.due_this_month_count`: count of non-paid payments whose effective due date falls in the current Israeli calendar month. The frontend must not infer this count from the reporting period month.
- `MonthBatchSummary.vat_mismatch_count`: count of the batch's payments whose stored turnover disagrees with their period's current VAT figure. Always returned (not conditional on the filter) — it is what lets a client hide the batches the `vat_mismatch` filter would empty, without fetching their rows first.

## Error codes

Codes follow `ADVANCE_PAYMENT.REASON` format. Registry: `docs/backend/error-codes.md`.

| Code | HTTP | Raised when |
|------|------|-------------|
| `ADVANCE_PAYMENT.CLIENT_RECORD_NOT_FOUND` | 404 | `client_record_id` does not exist |
| `ADVANCE_PAYMENT.FREQUENCY_NOT_SET` | 404 | Client has no configured `advance_payment_frequency` |
| `ADVANCE_PAYMENT.FREQUENCY_MISMATCH` | 409 | Request `period_months_count` does not match client's configured frequency |
| `ADVANCE_PAYMENT.CONFLICT` | 409 | Active payment already exists for `(client_record_id, period)` |
| `ADVANCE_PAYMENT.NOT_FOUND` | 404 | Payment ID not found for the given client |
| `ADVANCE_PAYMENT.RATE_INVALID` | 400 | VAT rate is zero when attempting reverse calculation (`advance_payment_calculator.py`) |
| `ADVANCE_PAYMENT.VAT_TURNOVER_NOT_FOUND` | 404 | Refresh found no VAT work item covering every month of the period |
| `ADVANCE_PAYMENT.VAT_NOT_FILED` | 409 | Refresh found only unfiled VAT returns and the request did not pass `confirm_pending` |
| `ADVANCE_PAYMENT.NOT_READY` | 400 | Close attempted while the readiness gate lists blocking issues |
| `OBLIGATION.LOCKED` | 400 | Any mutation on a submitted (closed) advance |
| `CLIENT_RECORD.CLOSED` | 409 | Client record is closed or frozen — cannot create payment. Raised by the shared client-eligibility guard; the message distinguishes closed from frozen, the code does not |

## Known issues

No open known issues.

## Resolved issues

- **F-005** (2026-06-05): `timing_status` and `paid_late` were computed from legacy `due_date`, ignoring `due_date_effective`. Fixed: response schemas now use `due_date_effective or due_date` for both derived values (`backend/app/advance_payments/schemas/advance_payment.py:50-62,153-155`).

## Decisions (preserved)

From `backend/docs/domain_decisions_v3.md` (v3.1, May 2026) and the archived legacy spec at `docs/archive/advance-payments-legacy.md`:

1. **`overdue` is computed, not stored.** Removed from the status enum. `timing_status` (`overdue | on_time`) is derived at read time from `due_date_effective or due_date` and `status`. ~~`paid_late` is similarly computed~~ — superseded in W3: lateness of the *close* is the stored `closed_late` fact, written once at the close (D-20); the computed `paid_late` field was removed from the contract.

2. **Turnover snapshot vs. live.** `turnover_amount` is a snapshot frozen on write. For pending/partial payments, `live_turnover` is fetched from `VatWorkItem` at read time via `TurnoverLookupRepository` when `turnover_amount is None`. No hard dependency on VAT report existing before advance payment.

   `available_turnover` is a *discovery* signal, not a second source of truth for the money: `calculated_amount`, `expected_amount`, and `status` are derived only from the stored `turnover_amount`. The advance payment is created before its period's VAT return exists, so a window where `turnover_amount is NULL` is unavoidable; `available_turnover` exists to show that the window can now be closed. Turning it into a snapshot is an explicit advisor action (the refresh command), never an automatic consequence of VAT filing.

   It was named `live_turnover` until 2026-07-20 and was rendered in the same UI slot as `turnover_amount`, which made a screen show one turnover while the entity held another. The rename to `available_turnover` plus the added `source` is what lets the UI render it as a pending action instead of a value.

   **Behaviour change in the same revision:** the read path now surfaces unfiled (`READY_FOR_REVIEW`) returns too, where it previously considered only `FILED` ones. A period whose VAT is still in review therefore stops counting as `missing_turnover` and starts advertising a `vat_pending` candidate. `missing_turnover` now means "nothing stored and nothing to snapshot", which is a different question from "nothing filed".

3. **Edit via drawer, not inline.** UI uses a drawer component for editing payments (UI decision, not enforced in backend).

4. **Overview grouped by month (collapsed by default).** Month-batch summaries are provided by `/overview/batches`. (UI decision.)

5. **`advance_rate` default from `LegalEntity`.** At creation time, if `advance_rate` is not passed, the service reads it from `LegalEntity.advance_rate`. The value is then snapshotted frozen on the record.

6. **`due_date_original` immutable; `due_date_effective` is overdue source of truth.** Architectural invariants INV-04 and INV-05. Enforced by SQLAlchemy event listeners.

7. **Workflow objects anchor on `client_record_id`, not `legal_entity_id`.** Invariant from `domain_decisions_v3.md` §1.

8. **Frequency independence.** `advance_payment_frequency` never derived from `vat_reporting_frequency`. Enforced in service and documented as INV-07.

9. **`TaxDeadline` removed.** All deadline lookups go through `TaxCalendarEntry`. New code must not use the `TaxDeadline` name or concept. (`domain_decisions_v3.md` §3.6)

10. **2216 rate-reduction requests are tracked as tasks, not as an advance-payments workflow** (2026-07-23). The request is submitted in שע"מ, outside the system; the system only follows it up. Model it as a regular task in the tasks module ("הגשת 2216 ללקוח X" + due date). When the authority approves, the advisor applies the new rate with the bulk `POST /clients/{id}/advance-payments/bulk-rate-update` action ("from period X onward"), which reprices pending rows and updates the legal entity's default — not by editing periods one by one. There is no 2216 entity, status, or endpoint.

11. **No client-facing notification for advance payments** (2026-07-26). Advance periods do not send reminders to the client: there is no `advance_payment_reminder` trigger, template, or send button, and none is to be added. Follow-up on unpaid periods is internal — the overdue filter, the `timing_status` badge, and the §190 interest indication. (`docs/advance-payments-product-roadmap.md` §Rejected)

12. **Refresh overwrites the edit form's turnover field unconditionally — no confirmation dialog** (2026-07-26). Pressing "קבע לפי מע״מ" in the detail form re-seeds the turnover input from the mutation result, discarding an unsaved manually typed turnover. This is deliberate, and the alternative was rejected: the command has already written the snapshot server-side by the time the form re-renders, so preserving the local draft would leave the form dirty and make the next Save send the pre-snapshot figure — resetting `turnover_source` to `manual` and destroying the fresh provenance (the MAT-74 defect). A confirmation dialog would also have to run *before* the mutation to mean anything, i.e. block the snapshot itself, and the only thing at risk is a draft the advisor discarded by pressing the refresh button on that same field. No other form field is touched by refresh. (`frontend/src/features/advancedPayments/components/panel/AdvancePaymentDetailView.tsx:91-98`; regression test `frontend/src/features/advancedPayments/hooks/useAdvancePaymentDetailForm.test.ts`)

## Future / planned

These items are explicitly **not yet implemented**. Do not describe as current behavior.

- **Remove legacy `AdvancePayment.due_date`.** The old `due_date` column coexists with `due_date_original` and `due_date_effective`. Plan: audit all consumers to use `due_date_effective`, then drop `due_date`. (`domain_decisions_v3.md` §3.3, §9)
- **Explicit due-date override endpoint.** No dedicated endpoint for updating `due_date_effective` exists yet. If added, it must enforce `due_date_override_reason`, permission checks, and must block updates to terminal-state records. (`domain_decisions_v3.md` §9, INV-09)
- **`turnover_source_vat_report_id` FK.** The legacy spec proposed storing the source VAT-report ID on the payment (`advance_payments_spec.md` §שינויים נדרשים). Partially addressed: `turnover_source` and `turnover_snapshot_at` now record *what kind* of source and *when*, but the specific `vat_work_item_ids` live only in the `advance_payment.turnover_refreshed` audit entry, not as a column.
- **Turnover drift warning.** Legacy spec proposed a ⚠ alert when the VAT report's turnover changed after the advance payment was recorded. Not implemented; `turnover_snapshot_at` is the field a drift check would compare against.
- **Bulk refresh from the cross-client overview.** The bulk endpoint is id-based and therefore screen-agnostic, but only the client tab calls it today. Wiring it to the overview needs a decision about what "all ready" means when a filter matches far more rows than the page shows.
- **`missing_turnover` blocking batch "mark ready".** Legacy spec proposed that `missing_turnover` blocks batch operations. Currently `missing_turnover` is a read signal only and does not block individual or batch writes.

## Historical notes

Legacy spec archived at `docs/archive/advance-payments-legacy.md`.

Historical source archived at `docs/archive/advance-payments-legacy.md`.
