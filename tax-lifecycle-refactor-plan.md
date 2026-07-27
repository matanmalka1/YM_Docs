## Scope

This file owns only:

- The cross-domain refactor plan for the tax lifecycle shared by VAT, advance payments, and annual reports.
- The evidence that motivated it, the decisions taken, and the decisions still open.

This file must not contain:

- Canonical current-state behavior for any of the three domains (see `docs/domains/vat.md`,
  `docs/domains/advance-payments.md`, `docs/domains/annual-reports.md`).
- Architecture rules or endpoint contracts (see `docs/architecture/*`, `docs/backend/architecture.md`).
- Behavior described as if already implemented.

Source of truth: tracking only — not source of truth for current behavior.

Created: 2026-07-27. Status: **planning, not started.** No code changes have been made.

---

# Tax Lifecycle Refactor — Plan

## 1. Why

VAT, advance payments, and annual reports are three views of one thing: a client owes a tax
obligation for a period, works it through a lifecycle, and settles it by a deadline. The code does
not say that anywhere. Each domain re-derives the shared concepts with its own vocabulary, its own
error codes, and its own timing rules, so the three drift against each other instead of composing.

The referenced-but-nonexistent "tax-lifecycle plan" in
`backend/app/advance_payments/services/advance_payment_service.py:243` is this file.

## 2. What already works — do not rebuild

A shared spine exists and is half-wired. This refactor **finishes** that pattern; it does not
introduce a new one.

| Shared mechanism | Location | Wired to |
|---|---|---|
| `TaxCalendarEntry` + `TaxCalendarMaterializationService` | `backend/app/tax_calendar/services/tax_calendar_materialization_service.py` | all three (NOT NULL FK on each root) |
| Client-eligibility guard `assert_client_record_is_active` + SQL twin `eligible_client_status_expr` | `backend/app/clients/guards/client_record_guards.py`, `backend/app/clients/repositories/client_active_scope.py` | all three |
| `is_vat_work_item_resolved` / `is_advance_payment_resolved` / `is_annual_report_resolved` | each domain's enum module | `tax_calendar_grouped_service.py` |
| `PeriodicObligationPlan` + per-type plan builders | `backend/app/common/obligation_plan.py` | VAT + advance payments only |
| Keyset-chunked, idempotency-keyed office-wide generation | `backend/app/advance_payments/services/advance_payment_service.py` (`bulk_generate_annual_schedules`) | advance payments only |

The guard/twin pair is the reference shape for every rule this plan consolidates: one Python
implementation, one SQL implementation, documented as required to change together.

## 3. Evidence

### 3.1 The lifecycle is three unrelated shapes

|  | VAT | Advance payments | Annual reports |
|---|---|---|---|
| Unit of work | period `YYYY-MM` | period `YYYY-MM` | `tax_year` |
| Lifecycle | 6-status graph with `VALID_TRANSITIONS` | 3 statuses **derived from money**, no transition graph | 7-status graph with `VALID_TRANSITIONS` |
| Terminal states | `filed`, `canceled` | `paid` | `closed`, `canceled` |
| Deadline fields | `due_date_original` / `due_date_effective` / `due_date_override_reason` | same three **plus** legacy `due_date` | `filing_deadline` + `deadline_type` + `custom_deadline_note` |
| Lateness | see §3.3 — inconsistent | computed `timing_status` | `/overdue` endpoint |
| Who creates rows | client onboarding only | onboarding + per-client `/generate` + office-wide `/bulk-generate` | obligation orchestrator only |

### 3.2 Duplicated rules

1. **Bi-monthly odd-month start rule — three implementations, three error codes.**
   - `backend/app/vat/services/vat_intake_service.py:33-45` → `VAT.INVALID_PERIOD_FOR_FREQUENCY`
   - `backend/app/advance_payments/services/advance_payment_service.py` `_validate_period_months_count` → `ADVANCE_PAYMENT.INVALID_PERIOD`
   - `backend/app/tax_calendar/services/tax_calendar_materialization_service.py:190` `_validate_period_alignment` → `TAX_CALENDAR.INVALID_PERIOD_ALIGNMENT`

   The calendar implementation is the real gate — both domains call it during materialization. The
   other two only fire earlier.

2. **`bimonthly_vat_period` and `bimonthly_advance_payment_period` are byte-identical.**
   `backend/app/common/period_utils.py:43-53` vs `:56-66`.

3. **Period parsing hand-rolled in ~12 call sites** (`int(period[:4])`, `period.split("-")[0]`)
   despite `parse_period_year` / `parse_period_month` existing in the same shared module.
   Sites include `vat_audit.py:16`, `vat_turnover_repository.py:42`,
   `vat_data_entry_invoices_service.py:103,112`, `vat_data_entry_invoice_update_service.py:148,159,162`,
   `vat_data_entry_common.py:116`, `vat_intake_service.py:40`,
   `tax_calendar_materialization_service.py:187`, `tax_calendar_entry.py:178`,
   `work_queue_metadata.py:41`, plus seed builders.

4. **"Advances paid for client + year" has two implementations that can disagree.**
   - SQL aggregate: `advance_payment_aggregation_repository.py:195` `sum_paid_by_client_year`
   - Python sum over a `page_size=10000` read: `annual_report_advances_summary_service.py:41-47`

   Consequence: `final_balance` is computed three different ways —
   `annual_report_query_service.py:150` (`tax_after_credits - advances_paid`),
   `annual_report_advances_summary_service.py:52` (same formula, other data source), and
   `annual_report_tax_service.py:127` (adds NI and VAT balance). The Python sum also silently
   undercounts past the page cap.

5. **Resolved-status set forked into SQL without a twin.**
   `backend/app/vat/repositories/vat_compliance_repository.py:88` hardcodes
   `notin_([FILED, CANCELED])` instead of calling `is_vat_work_item_resolved`. Adding a VAT status
   silently requires editing this repository. This is the exact failure mode the client-eligibility
   guard already solved with `eligible_client_status_expr`.

### 3.3 Flows not aligned

1. **Work-queue entry uses a different rule per domain.**
   - VAT (`backend/app/work_queue/items/tax_items.py` → `VatComplianceRepository.get_overdue_unfiled`)
     filters on **`period < current YYYY-MM`**, not on `due_date_effective`. This contradicts the
     documented invariant that `due_date_effective` is the overdue source of truth, and it means a
     VAT period **never appears in the queue before it is already late** — there is no upcoming
     window.
   - Advance payments and annual reports both use a due-date cutoff of
     `today + UPCOMING_WINDOW_DAYS`.
   - Advance-payment items are assembled in `backend/app/work_queue/items/billing_items.py`
     alongside charges, while VAT and annual reports live in `tax_items.py`. An advance payment is a
     tax obligation, not a billing artifact.

2. **Generation has three entry points and no rollover.**
   - VAT + advance payments: `backend/app/clients/services/client_onboarding_service.py:92-170`
     (`_sync_vat_work_items`, `_sync_advance_payments`), driven by `obligation_plan`.
   - Annual reports: `backend/app/actions/services/obligation_orchestrator.py`, which does **not**
     use `obligation_plan`.
   - Advance payments additionally have `POST /clients/{id}/advance-payments/generate` and the
     office-wide `POST /advance-payments/bulk-generate`.
   - **VAT has no bulk or office-wide generation at all.**
   - There is no scheduler. `backend/app/lifespan.py` runs signature-expiry and signature
     reconciliation only. Next year's obligations therefore exist only if someone onboards or edits
     a client, or manually runs advance bulk-generate.
   - `client_onboarding_service.py:100` imports the **private** `_years_to_generate` from
     `app.actions.services.obligation_orchestrator`, inside a function body — a layering inversion
     hidden from static import analysis.

3. **Three different coupling styles, one of them a cycle.**
   - VAT → advance payments: **pull**, explicit advisor action (refresh-turnover), frozen snapshot,
     one resolver (`TurnoverLookupRepository._resolve`) with a documented SQL twin. This is the
     model the other couplings should follow.
   - VAT → annual reports: **pull**, explicit advisor action (auto-populate) — but auto-populate
     does **not** invalidate persisted `tax_due` / `refund_due`, while manual income/expense
     mutations do. Readiness then reads a stale persisted value.
   - Advance payments → annual reports: **push**. `advance_payment_service.py:243` calls
     `AnnualReportTaxService.invalidate_tax_if_open` through a deferred function-body import,
     because `annual_reports` already imports `advance_payments` repositories. A real circular
     dependency, worked around rather than resolved.

4. **VAT balance is summed into an income-tax total.**
   `annual_report_tax_service.py:127` computes
   `total_liability = tax_after_credits + ni.total + (vat_balance or 0) - advances_paid`.
   `annual_report_pdf_builder.py:265` renders that number labelled `סה"כ חבות (מס + ביטוח לאומי)` —
   the label omits the VAT term the arithmetic includes. Already recorded as a known issue in
   `docs/domains/annual-reports.md`; still open.

### 3.4 Documentation drift found during this review

The domain docs are `Source of truth: mandatory`, so these are defects in their own right:

- `docs/domains/vat.md` lists `VAT.CLIENT_CLOSED` and `VAT.CLIENT_FROZEN` in the error table. Both
  are dead — `vat_intake_service.py:60-62` records that the VAT-local re-check was unreachable and
  removed once `VatClientContextService` adopted the shared guard.
- `docs/domains/annual-reports.md` known-issue list states that
  `AdvancePaymentService.bulk_mark_paid` does not invalidate persisted tax. It does, at
  `advance_payment_service.py:508`. The VAT-auto-populate half of that same known issue **is** still
  real (§3.3.3).

## 4. Target model

**One obligation, three specializations.** An obligation is
`(client_record, obligation_type, period | tax_year)` and is already anchored physically by
`TaxCalendarEntry` plus the FK each root carries.

Every obligation must answer these five questions the same way. The plan is organized around
closing the gaps in this table.

| # | Question | Canonical answer | Today |
|---|---|---|---|
| 1 | Is it owed? | `obligation_plan` | VAT + advance only; annual excluded |
| 2 | When is it due? | `due_date_effective` | annual uses `filing_deadline` |
| 3 | Is it resolved? | `is_*_resolved` **plus a SQL twin** | Python side only; VAT SQL forked |
| 4 | Is it late? | one rule over #2 and #3 | three different rules |
| 5 | May I act on this client? | `assert_client_record_is_active` | done |

## 5. Decisions taken

| # | Decision | Date |
|---|---|---|
| D-1 | VAT work-queue entry aligns to `due_date_effective` with the same upcoming window as advance payments and annual reports. Accepted consequence: VAT periods become visible in the queue **before** they are late, which is a visible product change. | 2026-07-27 |
| D-2 | The advance-payments → annual-reports dependency is inverted with an **explicit port** — a named interface that `annual_reports` implements and `advance_payments` depends on. An event/hook registry was rejected: it hides ordering and makes "who invalidated this figure" untraceable, against ADR-0003. | 2026-07-27 |
| D-3 | The unified bi-monthly alignment gate raises a **single `TAX_CALENDAR` error code**. `VAT.INVALID_PERIOD_FOR_FREQUENCY` and `ADVANCE_PAYMENT.INVALID_PERIOD` are retired for this condition. Accepted consequence: an API contract change — `openapi.json` + `generated.ts` regenerate, and both frontend message catalogs need updating. | 2026-07-27 |

## 6. Decisions still open

| # | Question | Why it blocks | Blocks |
|---|---|---|---|
| O-1 | **Rollover policy.** Should next year's obligations be opened by a scheduled job, by an advisor-triggered office-wide generate, or both? Today it is neither reliably — see §3.3.2. | Determines whether Phase 5 needs scheduler infrastructure that does not exist today, and what happens to a client whose frequency is configured mid-year. | Phase 5 |
| O-2 | Does `docs/domains/tax-lifecycle.md` (Phase 0) carry `Source of truth: mandatory`? Recommended yes — otherwise the five questions have no enforceable home and drift resumes. | Determines whether the three domain docs may keep restating the shared rules. | Phase 0 |
| O-3 | `ADVANCE_PAYMENT.INVALID_PERIOD` also covers "unsupported `period_months_count`", not only bi-monthly alignment. Under D-3, does that second condition also move to `TAX_CALENDAR`, or stay domain-owned? | Determines the exact scope of the D-3 contract change. | Phase 1 |
| O-4 | Advance payments have no transition graph — status is derived from money (§3.1). Should it stay that way under the unified model, or gain an explicit lifecycle? Deriving from money is defensible and currently documented; the question is whether "resolved" alone is enough shared vocabulary. | Determines how far Phase 3's unification goes. | Phase 3 |

## 7. Phases

Ordering is dependency-driven. Phase 0 gates everything. Phase 1 is safe to land alone.

### Phase 0 — Freeze the contract (documentation only)

No code. Deliverable: `docs/domains/tax-lifecycle.md`, owning:

- The five questions of §4 and the canonical answer to each.
- The obligation vocabulary: owed / open / resolved / late, and what each means per domain.
- The allowed coupling directions (see Phase 4).
- The generation and rollover policy (blocked on **O-1**).

Then: each of the three domain docs links to it and **stops restating** the shared rules. Fix the
two drift items in §3.4 in the same pass.

Nothing in Phases 1–6 starts before this file is agreed.

### Phase 1 — Delete provable duplication (no behavior change except D-3's error code)

| Item | Change | Files |
|---|---|---|
| 1.1 | One period-alignment gate. Delete the VAT and advance-payment local checks; the calendar materializer becomes the only gate, raising the single `TAX_CALENDAR` code per **D-3**. Resolve **O-3** first. | `vat_intake_service.py:33-45`, `advance_payment_service.py` `_validate_period_months_count`, `tax_calendar_materialization_service.py:190` |
| 1.2 | Delete `bimonthly_advance_payment_period`; rename the survivor to `bimonthly_period` and repoint callers. | `common/period_utils.py:43-66` |
| 1.3 | Replace hand-rolled period parsing with `parse_period_year` / `parse_period_month`. | ~12 sites listed in §3.2.3 |
| 1.4 | One implementation of "advances paid for client + year" — the SQL aggregate. Delete the Python sum. Then one definition of `final_balance`. | `annual_report_advances_summary_service.py:41-52`, `annual_report_query_service.py:150`, `annual_report_tax_service.py:127` |
| 1.5 | Add `*_resolved_expr()` SQL twins beside each `is_*_resolved`, documented as required-to-change-together. Use the VAT twin in place of the hardcoded status list. | three enum modules, `vat_compliance_repository.py:88` |

Frontend follow-up for 1.1: both VAT and advance-payments message catalogs plus the regenerated
contract (`openapi.json` → `generated.ts`).

**Done when:** each rule in §3.2 has exactly one implementation (or one Python + one SQL twin), and
`final_balance` returns the same number from every endpoint that publishes it.

### Phase 2 — Align lateness and the work queue (per D-1)

| Item | Change | Files |
|---|---|---|
| 2.1 | VAT work-queue entry moves from `period < now` to `due_date_effective` with the shared `UPCOMING_WINDOW_DAYS` cutoff, matching the other two. | `vat_compliance_repository.py` `get_overdue_unfiled`, `work_queue/items/tax_items.py` |
| 2.2 | Move `advance_payment_items` from `billing_items.py` to `tax_items.py`. | `work_queue/items/billing_items.py`, `work_queue/items/tax_items.py` |
| 2.3 | One urgency derivation shared by all three obligation types. | `work_queue/items/common.py` |

**Done when:** the three obligation types enter the work queue on one rule, and no work-queue module
re-derives a domain's resolved set or lateness.

### Phase 3 — Unify the deadline shape

Annual reports adopt `due_date_original` / `due_date_effective` / `due_date_override_reason`.
`filing_deadline` becomes `due_date_effective`; `deadline_type` stays, because standard/extended/
custom is a genuine annual-only regulatory choice, not a shape difference.

Same phase: drop the legacy `AdvancePayment.due_date` column — already on the advance-payments
roadmap — after auditing every consumer onto `due_date_effective`. Note that
`advance_payment_aggregation_repository.py` still coalesces to it in several expressions.

Payoff: the long-planned due-date-override endpoint (currently listed as future work in **both**
`docs/domains/vat.md` and `docs/domains/advance-payments.md`) gets built once for all three instead
of three times.

Depends on **O-4** for how far status unification goes.

**Done when:** one overdue rule reads one field name across all three, and two Alembic migrations
(annual adopt, advance drop) have landed with the enum-downgrade convention applied.

### Phase 4 — Fix the coupling directions (per D-2)

Rule to write into Phase 0's document and then enforce:

> Annual reports read from VAT and advance payments. VAT and advance payments never read or write
> annual reports.

| Item | Change | Files |
|---|---|---|
| 4.1 | Replace the deferred function-body import with an explicit port: an interface `advance_payments` depends on and `annual_reports` implements at composition time. | `advance_payment_service.py:243`, `annual_report_tax_service.py` `invalidate_tax_if_open` |
| 4.2 | Make VAT auto-populate invalidate persisted `tax_due` / `refund_due` exactly as manual line mutations do. Closes the still-real half of the annual-reports known issue. | `annual_report_vat_import_service.py` |

**Done when:** no module-level or function-level import crosses from `advance_payments` into
`annual_reports`, and every write path that changes a tax-calculation input invalidates the
persisted result.

### Phase 5 — One generation and rollover story

Blocked on **O-1**.

| Item | Change |
|---|---|
| 5.1 | Add `annual_obligation_plan(year)` to `common/obligation_plan.py` so annual reports answer question #1 the same way. |
| 5.2 | One `ObligationGenerationService` covering all three for a client + year. Collapses `_sync_vat_work_items`, `_sync_advance_payments`, and `obligation_orchestrator`, and removes the private `_years_to_generate` cross-domain import. |
| 5.3 | Office-wide bulk generation covers all three obligation types, **reusing** the existing keyset-chunk, per-chunk idempotency-key, non-atomic-with-reported-failures design. That design is sound — extend it, do not rewrite it. |
| 5.4 | Implement the rollover policy chosen in **O-1**. |

**Done when:** one service creates all three obligation types, one office-wide command covers all
three, and the rollover policy is implemented rather than incidental.

### Phase 6 — Fix the liability arithmetic

Drop `vat_balance` from `total_liability`; expose it as a separate informational field. Fix the PDF
label. Small and independent — can land any time after Phase 0.

Files: `annual_report_tax_service.py:127-129`, `annual_report_pdf_builder.py:265`, and the
`AnnualReportTaxCalculationResponse` schema (contract change → regenerate).

## 8. Risk summary

| Phase | Risk | Nature |
|---|---|---|
| 0 | none | documentation |
| 1 | low | internal, except D-3's contract + frontend catalog change |
| 2 | medium | **visible product change** — VAT appears in the queue earlier |
| 3 | medium-high | two schema migrations, wide consumer audit |
| 4 | medium | architectural; touches transaction boundaries |
| 5 | medium | consolidates three creation paths; needs O-1 |
| 6 | low | one arithmetic change + one label + contract regen |

## 9. Open items not in scope

Recorded so they are not silently absorbed:

- Unverified external tax constants (2026 NI ceiling, 2026 brackets, donation minimum) —
  `docs/domains/annual-reports.md` known issues. Needs the authority's circulars, not a refactor.
- Signature creation running inside the annual-report status-transition transaction — separate
  known issue.
- `AnnualReportDetail.updated_at` nullability — separate known issue.
