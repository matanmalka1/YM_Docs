## Scope
This file owns only:
- Canonical current-state documentation for the tax-calendar domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Tax Calendar

The tax-calendar domain owns the shared regulatory deadline ledger that all client workflow objects link to. `TaxCalendarEntry` stores one canonical deadline fact per obligation/period, `DeadlineRule` stores the versioned rule used to compute it, and the exposed routes either list grouped operational coverage over those entries or list/bootstrap the settings data behind them.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All paths confirmed present in `backend/openapi.json`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/tax-calendar/groups` | List grouped tax-calendar entries with linked open/done/overdue counts across VAT, advance payments, and annual reports |
| `GET` | `/api/v1/tax-calendar/groups/{tax_calendar_entry_id}/items` | List the client-level workflow items linked to one tax-calendar entry |
| `GET` | `/api/v1/settings/tax-calendar/rules` | List configured `DeadlineRule` rows |
| `GET` | `/api/v1/settings/tax-calendar/entries` | List raw `TaxCalendarEntry` rows, optionally filtered by year range |
| `GET` | `/api/v1/settings/tax-calendar/summary` | Return per-year entry counts plus warnings for missing expected coverage / registry fallback years |
| `POST` | `/api/v1/settings/tax-calendar/bootstrap` | Seed missing default rules and generate entries for a year range |

**Auth:** grouped routes allow `ADVISOR` and `SECRETARY`; settings routes are `ADVISOR`-only.  
Cite: `backend/app/tax_calendar/api/groups.py:13-20`, `backend/app/tax_calendar/api/groups.py:47-50`, `backend/app/tax_calendar/api/settings.py:25-29`, `backend/app/tax_calendar/api/settings.py:34-38`, `backend/app/tax_calendar/api/settings.py:48-52`, `backend/app/tax_calendar/api/settings.py:62-66`.

## Model & fields

**Table:** `tax_calendar_entries`  
Cite: `backend/app/tax_calendar/models/tax_calendar_entry.py:67-125`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | No | autoincrement |
| `obligation_type` | `ObligationType` enum | No | indexed; current generated values are `vat`, `advance_payment`, `annual_report` |
| `period` | `String(7)` | Yes | `YYYY-MM`; must be `NULL` for `annual_report` |
| `period_months_count` | int | Yes | `1` or `2` for periodic entries; `NULL` for `annual_report` |
| `tax_year` | int | No | must match `period[:4]` for periodic entries |
| `due_date` | date | No | shared regulatory due date, not per-client effective override |
| `deadline_rule_id` | int FK -> `deadline_rules.id` | No | `RESTRICT` on delete; indexed |
| `created_at` | datetime | No | set on insert |
| `updated_at` | datetime | Yes | set on update |

**Constraints / indexes:**
- `ck_tax_calendar_entry_period_nullability`: `period` is null only for `annual_report`. (`tax_calendar_entry.py:91-96`)
- `ck_tax_calendar_entry_months_count`: `period_months_count` is null only for `annual_report`; otherwise must be `1` or `2`. (`tax_calendar_entry.py:97-102`)
- Partial unique index `uq_tax_calendar_entry_periodic` on `(obligation_type, period, period_months_count)` for non-annual obligations. (`tax_calendar_entry.py:103-111`)
- Partial unique index `uq_tax_calendar_entry_annual` on `(obligation_type, tax_year)` for `annual_report`. (`tax_calendar_entry.py:112-119`)
- Index `idx_tax_calendar_entries_year_obligation` on `(tax_year, obligation_type)`. (`tax_calendar_entry.py:120-125`)

**Table:** `deadline_rules`  
Cite: `backend/app/tax_calendar/models/deadline_rule.py:34-77`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | No | autoincrement |
| `rule_type` | `DeadlineRuleType` enum | No | indexed |
| `due_day_of_month` | int | No | validated `1..31` |
| `offset_months` | int | No | non-negative; periodic due date shifts from period start month |
| `effective_from` | date | No | rule version start |
| `effective_to` | date | Yes | nullable = open-ended active rule |
| `description` | `String(255)` | Yes | optional note |
| `created_at` | datetime | No | set on insert |
| `updated_at` | datetime | Yes | set on update |

**Constraints / indexes:**
- `ck_deadline_rule_effective_range`: `effective_to >= effective_from` when present. (`deadline_rule.py:56-60`)
- `ck_deadline_rule_due_day_range`: `due_day_of_month BETWEEN 1 AND 31`. (`deadline_rule.py:61-64`)
- `ck_deadline_rule_offset_months_non_negative`: `offset_months >= 0`. (`deadline_rule.py:65-68`)
- `idx_deadline_rule_type_effective` on `(rule_type, effective_from)`. (`deadline_rule.py:69`)
- Partial unique index `uq_deadline_rule_open_ended` allows only one open-ended rule per `rule_type`. (`deadline_rule.py:70-76`)

## Enums / statuses

**`ObligationType`** (`backend/app/common/enums.py:50-60`)

| Value | Status |
|-------|--------|
| `vat` | Supported |
| `advance_payment` | Supported |
| `annual_report` | Supported |
| `national_insurance` | Reserved but intentionally unsupported in current tax-calendar generation/model validation |

**`DeadlineRuleType`** (`backend/app/common/enums.py:63-70`)

| Value | Used for |
|-------|----------|
| `vat_monthly` | Monthly VAT entries |
| `vat_bimonthly` | Bi-monthly VAT entries |
| `advance_monthly` | Monthly advance-payment entries |
| `advance_bimonthly` | Bi-monthly advance-payment entries |
| `annual_report` | Annual report entries |

There is no dedicated status enum in this domain. Grouped status is derived from linked domain objects:
- VAT is done only when `VatWorkItem.status == filed`. (`backend/app/tax_calendar/services/grouped_service.py:18-24`, `205-212`)
- Advance payment is done only when `AdvancePayment.status == paid`. (`grouped_service.py:18-24`, `205-212`)
- Annual report is done when status is one of `submitted`, `closed`, `canceled`. (`grouped_service.py:20-24`, `205-212`)

## Domain rules & invariants

- **One shared entry per obligation/period key.** `TaxCalendarEntry` is a regulatory fact shared across all clients, not a per-client record. Periodic uniqueness is `(obligation_type, period, period_months_count)`; annual uniqueness is `(obligation_type, tax_year)`. (`backend/app/tax_calendar/models/tax_calendar_entry.py:3-22`, `91-125`)
- **Annual vs periodic shape is enforced.** `annual_report` entries must have `period=NULL` and `period_months_count=NULL`; all other obligations require a valid `YYYY-MM` period and `period_months_count in (1, 2)`. (`tax_calendar_entry.py:127-137`, `147-183`)
- **`tax_year` must match the period year.** Periodic entries reject mismatched `period[:4]` vs `tax_year`. (`tax_calendar_entry.py:178-183`)
- **Rule compatibility is enforced.** A `TaxCalendarEntry` validates that its linked `DeadlineRule.rule_type` is allowed for the chosen `obligation_type`. `national_insurance` is rejected because no compatible `DeadlineRuleType` exists yet. (`tax_calendar_entry.py:45-52`, `157-161`, `185-200`)
- **DeadlineRule versioning is date-effective.** Active rule resolution always filters by `effective_from <= on_date <= effective_to/null` and picks the latest `effective_from`. (`backend/app/tax_calendar/repositories/deadline_rule_repository.py:38-50`)
- **Rule overlap is forbidden at service layer.** `has_overlapping_rule` treats `NULL effective_to` as open-ended infinity and rejects overlapping ranges for the same `rule_type`. (`backend/app/tax_calendar/services/deadline_rule_service.py:1-35`)
- **Generation is idempotent and does not mutate business objects.** `generate_for_year` / `generate_for_year_range` create missing shared entries only and skip existing keys. (`backend/app/tax_calendar/services/tax_calendar_entry_service.py:1-18`, `125-156`, `243-282`)
- **Periodic due-date calculation prefers the official tax-rules registry.** For VAT and advance-payment periodic rules, generation first asks `tax_rules.registry` for the official effective date; if unavailable or failing, it falls back to `DeadlineRule` (`due_day_of_month` + `offset_months`). (`tax_calendar_entry_service.py:6-18`, `85-90`; `backend/app/tax_calendar/integrations/tax_rules_registry.py:19-29`, `52-83`)
- **Annual due dates never use the registry.** Annual entries compute due date from `tax_year + 1`, shifted by `offset_months`, then clamped to `due_day_of_month`. Per-client annual deadline overrides live on `AnnualReport`, not on `TaxCalendarEntry`. (`tax_calendar_entry_service.py:15-18`, `92-95`)
- **Bi-monthly periods must start on odd months.** Materialization rejects `period_months_count=2` when the start month is even. (`backend/app/tax_calendar/services/materialization_service.py:197-203`)
- **Materialization links workflow objects to canonical entries and snapshots due dates.** `link_vat_work_item`, `link_advance_payment`, and `link_annual_report` assign or verify `tax_calendar_entry_id`; VAT and advance payments also backfill `due_date_original` / `due_date_effective` from the entry when empty. (`materialization_service.py:101-131`, `215-222`)
- **Mismatched existing links are rejected.** If a business object already has a different `tax_calendar_entry_id`, materialization raises `TAX_CALENDAR.LINK_CONFLICT`. (`materialization_service.py:215-222`)
- **Grouped calendar aggregation keys by regulatory period, not by due date.** Listing starts from `TaxCalendarEntry` rows, then attaches linked business objects per entry id. Overdue counts use each linked object's effective due date (`due_date_effective` or `filing_deadline`), not the shared regulatory date, while grouping stays anchored on the entry. (`backend/app/tax_calendar/services/grouped_service.py:75-132`, `188-212`)
- **Settings summary expects 37 generated entries per year.** Expected breakdown is 12 monthly VAT, 6 bi-monthly VAT, 12 monthly advance-payment, 6 bi-monthly advance-payment, and 1 annual report; summary/bootstrap warnings surface missing counts or registry-fallback years. (`backend/app/tax_calendar/services/settings_calendar_service.py:14-22`, `38-93`; `backend/app/tax_calendar/services/bootstrap.py:19-21`, `101-123`)
- **Year-range validation happens at the API edge.** Settings routes reject `tax_year_after > tax_year_before` with HTTP 400 before calling services. (`backend/app/tax_calendar/api/settings.py:17-23`, `39-45`, `53-59`, `67-74`)

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | HTTP | Raised when |
|------|------|-------------|
| `TAX_CALENDAR.NOT_FOUND` | 404 | Requested `tax_calendar_entry_id` does not exist in grouped item lookup |
| `TAX_CALENDAR.DEADLINE_RULE_MISSING` | 400 | Materialization needs a rule for the target obligation/year and none is active |
| `TAX_CALENDAR.INVALID_PERIOD_FREQUENCY` | 400 | Unsupported obligation/frequency combination was requested |
| `TAX_CALENDAR.INVALID_OBLIGATION_TYPE` | 400 | Unsupported obligation type was passed into materialization |
| `TAX_CALENDAR.INVALID_PERIOD` | 400 | `period` is missing or not in `YYYY-MM` format |
| `TAX_CALENDAR.INVALID_PERIOD_ALIGNMENT` | 400 | Bi-monthly period starts on an even month |
| `TAX_CALENDAR.INVALID_TAX_YEAR` | 400 | Tax year is not parseable or outside the accepted range |
| `TAX_CALENDAR.LINK_CONFLICT` | 409 | Existing `tax_calendar_entry_id` does not match the canonical entry for the object |

Cite: `backend/app/tax_calendar/services/grouped_items_service.py:27-30`, `backend/app/tax_calendar/services/materialization_service.py:133-142`, `176-212`, `215-222`.

## Decisions (preserved)

Still-true decisions preserved from `backend/docs/domain_decisions_v3.md` and related shared legacy notes:

1. **`TaxCalendarEntry` is a shared regulatory fact, not a per-client object.** The client workflow objects (`VatWorkItem`, `AdvancePayment`, `AnnualReport`) carry operational state; tax-calendar entries carry the shared deadline fact. (`domain_decisions_v3.md:28-37`, `102-125`)
2. **`TaxDeadline` is retired.** New code must use `TaxCalendarEntry` plus the per-object snapshot fields on the linked business domains, not the old `TaxDeadline` concept. (`domain_decisions_v3.md:197-219`)
3. **Grouping key is `(obligation_type, period, period_months_count)`, not `due_date`.** Identical due dates do not mean the same business period. (`domain_decisions_v3.md:306-319`)
4. **Overdue logic belongs to the linked workflow object's effective due date.** The main tax-calendar screen aggregates on `TaxCalendarEntry` but overdue/open urgency must be computed from `due_date_effective` or `filing_deadline`, not the shared regulatory due date. (`domain_decisions_v3.md:232-240`, `349-356`)
5. **Workflow objects anchor on `client_record_id`, not `legal_entity_id`.** Tax-calendar links therefore attach to those workflow objects, not directly to legal entities. (`domain_decisions_v3.md:43-68`)
6. **Generation creates workflow objects directly from tax-calendar entries; there is no `ClientTaxObligation` layer.** (`domain_decisions_v3.md:28-31`, `321-346`)
7. **`DeadlineRule` must stay versioned.** Historic due dates must remain stable when regulations change. (`domain_decisions_v3.md:36`, `86-100`)

## Future / planned

These items are explicitly not current behavior:

- **Remove legacy `AdvancePayment.due_date` after all consumers rely on `due_date_effective`.** This is a downstream-domain follow-up, not a tax-calendar schema change in the current module. (`backend/docs/domain_decisions_v3.md:37`, `151-154`, `392-395`)
- **Add an explicit due-date override endpoint only if product needs it.** If such an endpoint is added, it must enforce override reason, permissions, and terminal-state guards on the linked workflow domains. (`domain_decisions_v3.md:238-243`, `394-395`)
- **Possible future support for `national_insurance`.** The enum value is reserved, but current validation intentionally rejects it because no `DeadlineRuleType` or generation path exists yet. (`backend/app/common/enums.py:51-55`, `backend/app/tax_calendar/models/tax_calendar_entry.py:157-161`)

## Historical notes

No dedicated `backend/docs` tax-calendar legacy file exists for this domain. Historical rationale still lives in the shared decision log `backend/docs/domain_decisions_v3.md` and shared screen-spec notes `backend/docs/frontend_screen_spec.md`, but those shared files were not archived or pointer-rewritten in this domain-only pass to avoid cross-domain conflicts.
