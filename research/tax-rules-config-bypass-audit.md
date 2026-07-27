## Scope

This file owns only:

- A point-in-time audit of application paths that bypass, shadow, partially consume, or cannot
  consume `backend/tax_rules_config`.
- Evidence, runtime impact, and remediation boundaries for each finding.

This file must not contain:

- New tax-law decisions.
- A replacement for `docs/project/tax-rules-config.md` or the canonical domain documents.
- Claims that test fixtures are production tax-rule definitions.

Source of truth: research/audit only

# Tax Rules Config Bypass Audit

Audit date: 2026-07-27.

Canonical rule: `docs/project/tax-rules-config.md` requires official tax deadlines, VAT rates,
thresholds, obligation rules, and exceptional deferrals to remain in
`backend/tax_rules_config/`. Application code may materialize or snapshot rule output, but must not
define an alternate official rule.

## Method and classification

The audit compared:

1. The public registry API and every financial, calendar, obligation, statutory, and VAT-deduction
   value under `backend/tax_rules_config/app/tax_rules/`.
2. Every `tax_rules` import and adapter under `backend/app/`.
3. Matching numeric values, deadline constructors, eligibility decisions, frequency plans, silent
   exception handlers, model/repository defaults, demo seed paths, frontend constants, and
   standalone application artifacts.
4. Consumers of each suspect helper, so dead code and runtime-active behavior are distinguished.

Classifications used below:

- **Active bypass** — a runtime path defines or selects a tax rule without the registry.
- **Silent fallback** — registry failure or missing-year data silently selects a local rule.
- **Shadow rule** — a duplicate value or decision exists outside the package, even if the normal
  path currently supplies the canonical value.
- **Integration gap** — the package defines the rule, but the application data model or workflow
  cannot consume it.
- **Demo/prototype bypass** — non-production data generation or a standalone prototype defines its
  own tax rules. It can still create misleading data or be mistaken for a supported application.

Severity:

- **Critical** — can calculate, create, omit, or deadline a real obligation under the wrong rule.
- **High** — can produce a wrong tax value or makes an entire configured obligation unreachable.
- **Medium** — duplicate presentation/seed/default behavior can drift but is not the primary
  production calculation path.
- **Low** — hygiene or misleading API shape without a demonstrated wrong primary-path result.

Validation disposition:

- **Confirmed — active** — exercised by a production application path.
- **Confirmed — structural** — directly proven model/contract/integration gap; reachability does
  not depend on a hypothetical caller.
- **Dormant hazard** — concrete unsafe default or fallback exists, but all current production
  callers avoid or ignore it.
- **Non-production** — confirmed in seed/prototype code only.
- **Retracted** — reviewed candidate that does not meet the canonical-config bypass definition.

## Executive inventory

| ID | Finding | Classification | Severity | Disposition |
|---|---|---|---|---|
| TRC-001 | Tax-calendar registry failure silently falls back to `DeadlineRule` | Silent fallback | Critical | Confirmed — active |
| TRC-002 | Local hardcoded tax-calendar rules seed days 15 and 31 | Active bypass | High | Confirmed — active |
| TRC-003 | Periodic and annual calendar plans are locally enumerated | Shadow rule | High | Confirmed — active |
| TRC-004 | Annual-report deadlines contain local statutory and extension dates | Active bypass | Critical | Confirmed — active |
| TRC-005 | Annual-report integration imports package internals instead of the public registry | Active bypass | Medium | Confirmed — active |
| TRC-006 | Public-institution/control-holder deadline classification is local | Active bypass | High | Confirmed — active |
| TRC-007 | Configured VAT online deadline is ignored | Integration gap | High | Confirmed — active |
| TRC-008 | Common financial adapter converts every registry failure into `None` | Silent fallback | Low | Dormant hazard |
| TRC-009 | Advance-payment VAT rate falls back to env/18% and is frozen at import | Shadow rule | Low | Dormant hazard |
| TRC-010 | Resident credit points have nested 2.25 fallbacks | Silent fallback | High | Confirmed — active |
| TRC-011 | Tax-engine default points are selected by process year, not report year | Shadow rule | Low | Dormant hazard |
| TRC-012 | NI engine falls back to the latest supported year's brackets | Silent fallback | Critical | Confirmed — active |
| TRC-013 | NI engine defaults omitted tax year to 2024 | Shadow rule | Low | Dormant hazard |
| TRC-014 | NI applicability is locally inferred from filing type | Active bypass | Critical | Confirmed — active |
| TRC-015 | Application never calls the obligation resolver | Integration gap | Critical | Confirmed — structural |
| TRC-016 | VAT and advance obligation schedules are locally resolved | Active bypass | Critical | Confirmed — active |
| TRC-017 | Bulk advance eligibility is “frequency is not null” | Active bypass | High | Confirmed — active |
| TRC-018 | VAT resolver has a hidden legacy monthly fallback | Silent fallback | Critical | Confirmed — active |
| TRC-019 | VAT applicability is repeated across validation, reminders, intake, and seed validation | Shadow rule | Medium | Confirmed — active |
| TRC-020 | Legal-entity data cannot represent the canonical obligation profile | Integration gap | Critical | Confirmed — structural |
| TRC-021 | Application enums/calendar cannot represent most configured obligations | Integration gap | Critical | Confirmed — structural |
| TRC-022 | Frontend hardcodes the exceptional-invoice threshold | Shadow rule | Medium | Confirmed — active |
| TRC-023 | VAT invoice persistence defaults to full deduction | Shadow rule | Low | Dormant hazard |
| TRC-024 | VAT derived-field helper defaults its tax year to 2026 | Shadow rule | Low | Dormant hazard |
| TRC-025 | Demo VAT seed duplicates rates, deductions, and threshold | Demo/prototype bypass | Medium | Non-production |
| TRC-026 | Demo annual-report seed hardcodes resident credit points | Demo/prototype bypass | Medium | Non-production |
| TRC-027 | Osek-patur warning threshold candidate | Product policy | None | Retracted |
| TRC-028 | Standalone tax-advances HTML has a separate tax configuration | Demo/prototype bypass | Medium | Non-production |
| TRC-029 | Client creation forces an advance-payment obligation | Active bypass | Critical | Confirmed — active |
| TRC-030 | Excel import invents entity and reporting-frequency defaults | Silent fallback | Critical | Confirmed — active |
| TRC-031 | Annual obligations are generated without registry applicability | Shadow rule | Medium | Confirmed — active |
| TRC-032 | Requested annual filing type is not matched to the client's entity | Active bypass | Critical | Confirmed — active |
| TRC-033 | Annual primary form selection is locally mapped | Shadow rule | High | Confirmed — active |
| TRC-034 | Canonical Form 6111 attachment rule is not consumed | Integration gap | High | Confirmed — structural |
| TRC-035 | Canonical small-business annual rule is unreachable | Integration gap | High | Confirmed — structural |
| TRC-036 | Creation preview treats missing VAT frequency differently from runtime | Silent fallback | High | Confirmed — active |
| TRC-037 | Registry corrections do not refresh existing calendar entries | Integration gap | High | Confirmed — structural |
| TRC-038 | Registry-derived dates are attributed to a local DB rule | Shadow rule | High | Confirmed — active |
| TRC-039 | Workflow deadline snapshots do not receive registry corrections | Integration gap | High | Confirmed — structural |
| TRC-040 | Existing standard annual deadlines do not receive official extensions | Integration gap | High | Confirmed — structural |
| TRC-041 | Stored osek-patur ceiling is frozen to the client-creation year | Shadow rule | Medium | Confirmed — active |
| TRC-042 | Advance-payment profile field names do not match the canonical contract | Integration gap | Critical | Confirmed — structural |

## Detailed findings

### TRC-001 — broad exception handling activates a local calendar

Evidence:

- `backend/app/tax_calendar/integrations/tax_rules_registry.py:10-16` catches every exception while
  importing the package and marks it unavailable.
- `backend/app/tax_calendar/integrations/tax_rules_registry.py:64-82` catches every runtime
  exception, logs a warning, and returns `None`.
- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:92-96` uses
  `get_registry_due_date(...) or base`.

Impact: missing package installation, a programming error, malformed registry data, or an
unsupported year all collapse into the same local deadline. The system continues with a plausible
but unofficial date, so the failure is difficult to detect and exceptional deferrals can be lost.

Required boundary: only an explicitly supported “no calendar for this year” result may be handled
as product behavior. Import errors and unexpected exceptions must not become tax dates.

### TRC-002 — hardcoded application deadline rules

Evidence:

- `backend/app/tax_calendar/services/tax_calendar_bootstrap_service.py:22-39` defines
  `DEFAULT_EFFECTIVE_FROM = 2023-01-01` and rules for VAT/advances on day 15 plus an annual rule on
  day 31.
- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:83-101` calculates dates from
  those DB rules.

Impact: the application contains a second official-rule store without `source_ids`. The annual rule
produces 31 May for every entity and cannot represent company, small-business, or annual override
dates. Periodic rules lose official holiday/exception changes whenever TRC-001 is activated.

### TRC-003 — local obligation calendar shape

Evidence:

- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:59-74` independently enumerates
  12 monthly and six bi-monthly VAT and advance-payment periods.
- `backend/app/tax_calendar/services/tax_calendar_bootstrap_service.py:23` hardcodes 37 expected
  entries per year.
- `backend/app/tax_calendar/services/tax_calendar_settings_calendar_service.py:14-22` repeats the
  expected per-obligation counts.

Impact: adding, removing, or changing a canonical obligation does not change generated coverage.
The summary can report “complete” while configured annual exempt declarations, PCN874, withholding,
and national-insurance obligations are absent.

### TRC-004 — annual-report dates defined in the application

Evidence:

- `backend/app/annual_reports/annual_report_deadlines.py:60-91` defines 31 July, 30 June, and
  fallback 29 May application-side.
- `backend/app/annual_reports/annual_report_deadlines.py:94-96` defines the representative extension
  as 31 January two years after the tax year.
- Runtime consumers include
  `backend/app/annual_reports/services/annual_report_create_service.py:97-105` and
  `backend/app/annual_reports/services/annual_report_status_service.py:137-143,249-255`.

Impact: these values bypass year-specific registry data and source metadata. The 29 May fallback
also conflicts with the package's individual default of 31 May.

### TRC-005 — annual reports bypass the public registry API

Evidence:

- `backend/app/annual_reports/annual_report_deadlines.py:6-12` imports
  `tax_rules.obligations.annual_reports.ANNUAL_REPORT_RULES_V2` and `tax_rules.types.EntityType`.
- The supported public entry point is `tax_rules.registry.get_annual_report_rule`.

Impact: application code is coupled to internal package layout and reimplements selection,
tax-year-specific lookup, fallback, and datetime conversion. Future registry behavior, validation,
or override handling can be missed while the import still succeeds.

### TRC-006 — unsupported filing types are assigned a local corporate rule

Evidence:

- `backend/app/annual_reports/annual_report_deadlines.py:21-30` states that `CONTROL_HOLDER` and
  `PUBLIC_INSTITUTION` have no matching `tax_rules_config` entity and assigns both the corporate
  deadline.

Impact: this is a domain/tax classification decision outside the canonical package and without
official sources. The application cannot distinguish form 1215/public-institution behavior from a
company rule.

### TRC-007 — configured VAT online deadline is not consumed

Evidence:

- `tax_rules.registry.get_vat_online_extended_deadline_day(year)` exposes the configured annual
  deadline day.
- `backend/app/vat/vat_report_queries.py:15-31` accepts `submission_method` but sets statutory,
  submission, and extended deadlines to the same stored date.

Impact: the public API can label a value `extended_deadline` while never applying the configured
online deadline. The unused parameter makes the behavior look method-aware when it is not.

### TRC-008 — financial registry failures are erased

Evidence:

- `backend/app/common/integrations/tax_rules_financials.py:4-10` catches every exception and returns
  `None`.

Impact: callers cannot distinguish unsupported year, missing key, broken install, or code defect.
This is the enabling path for TRC-009.

### TRC-009 — alternate VAT-rate configuration for advances

Evidence:

- `backend/app/advance_payments/advance_payment_constants.py:10-17` reads the current-year registry
  rate once at module import, otherwise reads `ADVANCE_PAYMENT_VAT_RATE`, defaulting to `0.18`.
- No production consumer of `ADVANCE_PAYMENT_VAT_RATE` was found at audit time.

Impact: currently dormant shadow configuration. If reused, it selects by process date rather than
the payment period and permits an environment variable to override the official yearly config.

### TRC-010 — resident credit points silently become 2.25

Evidence:

- `backend/app/annual_reports/integrations/tax_rules_registry.py:6-12` catches every exception and
  returns `Decimal("2.25")`.
- `backend/app/annual_reports/annual_report_tax_engine.py:23-28` wraps that helper in another broad
  fallback to `2.25`. Under the current helper implementation, the inner fallback normally prevents
  registry exceptions from reaching the outer one; the outer fallback remains reachable only for
  a conversion/unexpected return-value failure.

Impact: unsupported years and broken registry access produce a valid-looking tax calculation rather
than an explicit unsupported-year error.

### TRC-011 — dormant default credit points use the server year

Evidence:

- `backend/app/annual_reports/annual_report_tax_engine.py:23-28` resolves the default using
  `israel_today().year`.
- `backend/app/annual_reports/annual_report_tax_engine.py:54-61` captures the result as a Python
  default argument for every call that omits `credit_points`.

Impact: a historical or future report can inherit the server startup year's default. The value is
also frozen until the process restarts. Deep reachability check: both current production callers
pass `credit_points` explicitly
(`backend/app/annual_reports/api/annual_report_routes_financials.py:58-62`,
`backend/app/annual_reports/services/annual_report_tax_service.py:108-115`), so the unsafe function
default is presently dormant.

### TRC-012 — unsupported NI year uses another year's brackets

Evidence:

- `backend/app/annual_reports/annual_report_ni_engine.py:38-42` catches `KeyError` and asks for
  `max(get_supported_tax_years())`.

Impact: a 2027 calculation can silently use 2026 ceilings and rates. This is a direct cross-year tax
calculation error.

### TRC-013 — dormant omitted NI tax year means 2024

Evidence:

- `backend/app/annual_reports/annual_report_ni_engine.py:26-30` declares `tax_year: int = 2024`.

Impact: any caller omitting the year receives a historical rule without warning. Deep reachability
check: the only production caller currently passes `report.tax_year` explicitly
(`backend/app/annual_reports/services/annual_report_tax_service.py:116-118`), so this is a dormant
API hazard rather than a current wrong-result path. A tax-year input should still be required
because the package is year-versioned.

### TRC-014 — NI applicability bypasses `NATIONAL_INSURANCE_RULES`

Evidence:

- `backend/app/annual_reports/annual_report_ni_engine.py:11-14` defines local exempt filing types.
- `backend/app/annual_reports/annual_report_ni_engine.py:35-36` returns zero solely from that set.
- Canonical applicability uses `entity_type`, `btl_status`, and `has_employees`.

Impact: annual filing type is not an adequate substitute for national-insurance status. Employer
102 and self-employed/not-self-employed distinctions cannot be expressed.

### TRC-015 — canonical obligation resolver has no application consumer

Evidence:

- No call to `tax_rules.registry.get_obligations`, `resolve_obligation_rules`, or construction of
  `ClientTaxProfile` exists under `backend/app/`.

Impact: all configured `VAT_RULES`, `INCOME_TAX_ADVANCE_RULES`, `NATIONAL_INSURANCE_RULES`, and
`WITHHOLDING_RULES` are effectively documentation/package-test data rather than application
authority.

### TRC-016 — local VAT and advance schedule resolver

Evidence:

- `backend/app/common/obligation_plan.py:12-62` decides exemption, monthly/bi-monthly frequency,
  employee exclusion, period starts, and months count.
- Consumers:
  `backend/app/clients/services/client_onboarding_service.py:92-176`,
  `backend/app/clients/services/client_impact_preview_service.py:24-86`, and
  `backend/app/seed/builders/demo/advance_payments.py:129-141`.

Impact: changes to canonical obligation scope do not affect onboarding, preview, or demo generation.

### TRC-017 — bulk advance eligibility ignores obligation rules

Evidence:

- `backend/app/advance_payments/repositories/advance_payment_generation_repository.py:38-56`
  selects active clients whenever `advance_payment_frequency IS NOT NULL`.
- `backend/app/advance_payments/services/advance_payment_service.py:170-188` derives months count
  directly from the stored enum.

Impact: eligibility is a DB-null check, not a canonical tax-profile resolution. Any invalid or newly
unsupported entity/frequency combination remains eligible.

### TRC-018 — hidden legacy monthly VAT fallback

Evidence:

- `backend/app/vat/vat_type_resolver.py:6-18` returns `MONTHLY` when a non-exempt client lacks
  `vat_reporting_frequency`; the docstring explicitly calls it a legacy default.

Impact: missing tax configuration creates monthly obligations rather than surfacing a data error.
It conflicts with both the no-legacy and no-hidden-fallback project rules.

### TRC-019 — VAT applicability is repeatedly redefined

Evidence:

- `backend/app/clients/client_create_policy.py:17-41`
- `backend/app/clients/schemas/client_create_validation.py:36-100`
- `backend/app/vat/services/vat_intake_service.py:30-45`
- `backend/app/notifications/services/notification_policy_service.py:222-225`
- `backend/app/seed/validator.py:58-67`

Impact: these paths independently decide who is exempt, whether company/exempt combinations are
valid, period alignment, or whether reminders are allowed. They can drift from each other and from
`VAT_RULES`; the notification check, for example, checks `OSEK_PATUR` but not every effective
no-VAT profile.

### TRC-020 — the application cannot construct the canonical profile

Evidence:

- `tax_rules.types.ObligationScope` uses `requires_pcn874`, `has_employees`,
  `has_withholding_file`, `has_representative`, and `btl_status`.
- `backend/app/legal_entities/models/legal_entity.py:24-48` stores entity type, VAT frequency,
  advance frequency/rate, and financial fields, but none of those five obligation inputs.
- The package README also recommends tax file identifiers and BTL advance information that are not
  represented on this model.

Impact: TRC-015 cannot be corrected only by replacing a helper call. The persistence and API
contracts cannot express enough facts to resolve PCN874, withholding, employer BTL, self-employed
BTL, or representative-scoped rules.

### TRC-021 — application obligation vocabulary is incomplete

Evidence:

- `backend/app/common/enums.py:50-70` supports VAT, advances, annual reports, and a reserved
  national-insurance value. Deadline rule types cover only VAT, advances, and one annual type.
- `tax_rules.types.ObligationKind` additionally defines exempt annual VAT declaration, PCN874,
  self-employed BTL advances, employer 102, monthly withholding 102, annual 126, and annual 856.
- `backend/app/tax_calendar/models/tax_calendar_entry.py:151-175` explicitly rejects national
  insurance and has no representation for the other kinds.

Impact: most canonical obligations cannot be materialized, grouped, linked, or reported by the
application even when the package resolves them.

### TRC-022 — frontend exceptional-invoice threshold

Evidence:

- `frontend/src/features/vatReports/constants/vatConstants.ts:119-121` defines `25_000` and embeds
  it in user-facing text.
- The canonical value is yearly
  `exceptional_invoice_threshold_ils`; backend invoice classification reads it by period year.

Impact: after a threshold change, backend classification and frontend explanation can disagree.

### TRC-023 — persistence API can default to full VAT deduction

Evidence:

- `backend/app/vat/models/vat_invoice.py:94-98` defaults `deduction_rate` to `1.0000`.
- `backend/app/vat/repositories/vat_invoice_repository.py:45-73` defaults its create argument to
  `1.0`.

Impact: the primary invoice service passes a registry-derived rate at
`backend/app/vat/services/vat_data_entry_invoices_service.py:170-185`, and the only direct model
construction is demo seed code that also supplies the field. No current production write was found
that activates the default. It remains a dormant persistence hazard: a future direct
repository/model creation can silently claim full input deduction.

### TRC-024 — VAT derived fields default to tax year 2026

Evidence:

- `backend/app/vat/vat_data_entry_common.py:63-71` declares `year: int = 2026`.
- The primary create/update paths pass the work-item period year, so no current primary-path error
  was found.

Impact: a future direct caller can use 2026 deduction/threshold context accidentally. The misleading
default is unnecessary because every valid invoice belongs to a known period. Both current
production callers pass the work-item period year, so the default is presently dormant.

### TRC-025 — demo VAT seed is an independent tax engine

Evidence:

- `backend/app/seed/builders/demo/vat.py:40-54` duplicates deduction rates.
- `backend/app/seed/builders/demo/vat.py:357-361` uses VAT rate `0.18`.
- `backend/app/seed/builders/demo/vat.py:395-405` uses exceptional threshold `25_000`.

Impact: demo data can disagree with the selected period year and canonical category rules. It also
normalizes every listed vehicle-related category to two-thirds despite the package warning that
vehicle classification requires case-specific review.

### TRC-026 — demo credit points are hardcoded

Evidence:

- `backend/app/seed/builders/demo/reports.py:562-577` stores `2.25` for resident credit-point rows.

Impact: seeded historic/future reports do not obtain the year-specific default from the registry.

### TRC-027 — retracted: local 80% osek-patur warning policy

Evidence:

- `backend/app/vat/vat_constants.py:60` defines `OSEK_PATUR_CEILING_WARNING_RATE = 0.80`.
- `backend/app/vat/vat_data_entry_common.py:102-128` combines it with the canonical annual ceiling.

Disposition: retracted as a tax-rules-config bypass. This is an internal product-warning threshold,
not an official ceiling/rate/deadline, and no code or canonical documentation was found presenting
80% as a statutory value. It remains eligible for product-policy documentation, but is not counted
as a confirmed bypass.

### TRC-028 — standalone HTML tax configuration

Evidence:

- `tax-advances-system.html:905-915` defines tax year 2026, VAT 18%, interest/linkage rows, and a
  45% excess-expense rate.

Impact: this file is outside both deployable repositories and no production import was found. It is
therefore classified as a prototype, not a confirmed production path. It remains a complete
alternate tax configuration that can drift or be mistaken for supported behavior.

### TRC-029 — client creation forces an advance-payment obligation

Evidence:

- `tax_rules.types.ReportingFrequency` includes `NONE`, and canonical advance rules apply only when
  the profile explicitly carries monthly or bi-monthly advance frequency.
- `backend/app/common/enums.py:43-48` gives `AdvancePaymentFrequency` only monthly and bi-monthly
  values.
- `backend/app/clients/schemas/client_create_validation.py:71-72` rejects a missing frequency.
- `backend/app/clients/client_constants.py:70` presents that rejection as “advance-payment
  frequency is required.”

Impact: the application cannot create a supported business client who has no income-tax advance
obligation. It forces the user to select a frequency, after which onboarding and bulk generation
treat the client as liable. This changes “no configured obligation” into a mandatory obligation
outside the canonical resolver.

### TRC-030 — Excel import invents tax classifications and frequencies

Evidence:

- `backend/app/clients/client_constants.py:41-43` defaults an imported row to `OSEK_MURSHE`,
  bi-monthly VAT, and bi-monthly advances.
- The same file labels entity type and both frequencies optional in the import template at
  `backend/app/clients/client_constants.py:18-27`.
- `backend/app/clients/services/client_excel_service.py:117-141` applies the parsed/defaulted values
  and creates the client.

Impact: an omitted spreadsheet cell does not remain an explicit data gap. It creates a tax entity
classification and two recurring obligation schedules without registry resolution or user
confirmation.

### TRC-031 — annual generation timing and applicability are locally owned

Evidence:

- `backend/app/actions/services/obligation_orchestrator.py:33-40` chooses the current year and,
  beginning in October, the next year using local constant
  `CLIENT_OBLIGATION_NEXT_YEAR_START_MONTH`.
- `backend/app/actions/services/obligation_orchestrator.py:43-46,95-119` maps every recognized
  entity to a filing type and creates one annual report for each selected year.
- It does not call `get_annual_report_rule` or `get_obligations`, and it does not check whether an
  applicable non-attachment annual rule exists for the profile/year.

Impact: report creation timing, year coverage, and applicability are application policy. Every
currently auto-created business entity does happen to match a general annual rule, so no present
wrong-entity omission was demonstrated. The confirmed bypass is that registry effective versions
and absence of a supported annual year cannot prevent automatic creation.

### TRC-032 — annual filing type is trusted independently of legal entity

Evidence:

- `backend/app/annual_reports/services/annual_report_create_service.py:53-70` loads the
  `ClientRecord`, validates only that the submitted `client_type` is a known enum value, and does
  not compare it with the linked `LegalEntity.entity_type`.
- `backend/app/annual_reports/services/annual_report_create_service.py:95-105` then uses that
  caller-selected type for form and deadline selection.

Impact: a caller can create a corporate-form/deadline report for an osek client or an individual
report for a company. The registry's entity scope is bypassed even when its deadline values are
available.

### TRC-033 — primary annual form selection is a local rule table

Evidence:

- `backend/app/annual_reports/annual_report_constants.py:8-22` maps filing types to forms 1301,
  1214, and 1215.
- `backend/app/annual_reports/services/annual_report_create_service.py:95` uses that table.
- Canonical `AnnualReportRule` already carries the applicable `form` and entity scope.

Impact: form selection can drift independently from deadline selection. In particular, form 1215
and partnership/control-holder classifications are application-only extensions without canonical
rule objects or `source_ids`.

### TRC-034 — configured Form 6111 attachment rule is unused

Evidence:

- `backend/tax_rules_config/app/tax_rules/obligations/annual_reports.py:93-113` defines Form 6111 as
  an attachment scoped to osek-murshe/company profiles. Its notes say applicability depends on a
  specific obligation and recommend storing `requires_form_6111`.
- `backend/app/annual_reports/services/annual_report_schedule_service.py:45-68` generates Schedule
  A, Form 1504, and flag-driven schedules, but never evaluates the 6111 rule.
- Neither `ObligationScope` nor `LegalEntity` has a `requires_form_6111` field, and no application
  consumer of that identifier was found.

Impact: the package contains attachment metadata but does not encode the discriminator needed to
resolve it, and the application never consumes even the available entity/form metadata. Form 6111
exists in the application enum and can be added manually, but canonical config has no effect.

### TRC-035 — small-business annual rule cannot be selected

Evidence:

- `backend/tax_rules_config/app/tax_rules/obligations/annual_reports.py:49-69` defines a distinct
  small-business 1301 rule with an earlier deadline.
- Both that rule and the general individual rule share the same entity scope; the package profile
  has no small-business discriminator.
- `tax_rules.policy.resolve_annual_report_rule` returns the first matching non-attachment rule, so
  the general individual rule wins before the small-business rule.
- `backend/app/annual_reports/annual_report_deadlines.py:41-45` repeats the same first-match
  selection over the internal rule tuple.

Impact: the rule exists in canonical config but is unreachable through both the public resolver and
the application's direct integration. Every osek profile receives the general individual rule.

### TRC-036 — preview and runtime disagree on missing VAT frequency

Evidence:

- `backend/app/clients/services/client_impact_preview_service.py:34-42` treats
  `vat_reporting_frequency is None` as exempt and predicts zero VAT work items.
- `backend/app/vat/vat_type_resolver.py:12-18` treats a missing frequency for a non-exempt entity as
  monthly through its legacy fallback.

Impact: the pre-creation/update impact preview can promise no VAT obligations while runtime intake
or later resolution treats the same incomplete profile as monthly. Neither path uses the canonical
profile validator to reject the ambiguity.

### TRC-037 — registry changes do not refresh materialized calendar entries

Evidence:

- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:132-163` implements
  `_create_entry_if_missing`; an existing key immediately returns `False` without comparing or
  updating its due date.
- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:250-282` describes generation as
  idempotent and loads all existing keys before asking the registry for missing entries.

Impact: if an official deferral is added to `tax_rules_config` after a year was bootstrapped,
rerunning generation leaves the old application date untouched. The registry becomes authoritative
only at first materialization, not for later official corrections affecting still-open obligations.

Required boundary: historic filed/closed snapshots may intentionally remain immutable, but open
regulatory facts need an explicit reconciliation operation with audit/provenance rather than
create-only idempotency.

### TRC-038 — registry output is attributed to a local deadline rule

Evidence:

- `backend/app/tax_calendar/services/tax_calendar_entry_service.py:182-197` resolves a local
  `DeadlineRule`, obtains a possibly registry-derived `due`, and stores the local
  `deadline_rule_id`.
- `TaxCalendarEntry` has no tax-rules rule id, package version, source id, registry column, or
  override identifier.

Impact: a date changed by a package exception can still appear linked to the generic local “day 15”
rule. The stored provenance cannot explain whether the result came from the official calendar, an
exceptional override, or the local fallback. This also prevents precise reconciliation for
TRC-037.

### TRC-039 — client workflow snapshots do not receive corrected dates

Evidence:

- `backend/app/tax_calendar/services/tax_calendar_materialization_service.py:101-122` assigns
  `due_date_original` and `due_date_effective` for VAT and advance payments only when each field is
  `None`.
- Existing linked objects with populated snapshots are not compared with a corrected
  `TaxCalendarEntry`.
- No registry/calendar reconciliation service for open VAT work items or advance payments was
  found.

Impact: even if a shared calendar entry is corrected manually or by a future fix to TRC-037,
already-linked open workflow items retain the former deadline. User-facing overdue status,
reminders, reports, and work-queue urgency can therefore continue bypassing the corrected
canonical date.

### TRC-040 — existing annual reports do not receive new official extensions

Evidence:

- `backend/app/annual_reports/services/annual_report_create_service.py:95-121` resolves and stores
  `filing_deadline` only when the report is created.
- `backend/app/annual_reports/services/annual_report_status_service.py:126-143,239-262` recalculates
  it only as a side effect of an explicit deadline-type/submission-method update.
- No service or background job reconciles open `STANDARD` annual reports after
  `tax_rules_config` receives a tax-year-specific extension.

Impact: a report created before an official extension remains on its former date indefinitely.
Adding the correct override to the canonical package does not correct existing open reports,
overdue calculations, reminders, or grouped tax-calendar output.

### TRC-041 — stored osek-patur ceiling is creation-year data presented as current

Evidence:

- `backend/app/clients/client_create_policy.py:26-32` resolves the ceiling using
  `israel_today().year`.
- `backend/app/clients/services/client_create_service.py:173-196` stores that value on
  `LegalEntity.vat_exempt_ceiling`.
- No annual refresh path for the stored column was found.
- `frontend/src/features/clients/components/detail/ClientInfoSection.tsx:58-59` and
  `frontend/src/features/clients/components/form/ClientEditFormSections.tsx:189` present the stored
  value as a system value.
- Actual invoice enforcement separately reads the registry by invoice period year in
  `backend/app/vat/vat_data_entry_common.py:116-128`.

Impact: after the year changes, the UI can display the old official ceiling while enforcement uses
the new one. The same rule has two time semantics: creation-year snapshot for client display and
period-year lookup for validation.

### TRC-042 — advance-payment field names do not match `ClientTaxProfile`

Evidence:

- The canonical resolver and validator read `income_tax_advance_frequency` and
  `income_tax_advance_rate` in
  `backend/tax_rules_config/app/tax_rules/policy.py:44-53` and
  `backend/tax_rules_config/app/tax_rules/validations.py:38-45`.
- The application persists and transports the corresponding facts as
  `advance_payment_frequency` and `advance_rate` in
  `backend/app/legal_entities/models/legal_entity.py:37-44` and the client request/response
  contracts.
- No explicit adapter mapping those names into a `ClientTaxProfile` was found.

Impact: even after adding a call to `get_obligations`, passing an application entity/profile dict
directly would make the canonical resolver see no advance frequency and emit no advance
obligations. The mismatch is silent because the resolver uses `.get(...)` rather than requiring the
keys, and the validator would not require a rate for the same reason.

## Explicit exclusions and non-findings

The following were reviewed but are not counted as bypasses:

- Hardcoded dates and amounts in `backend/tests/` when they are fixtures or expected values only.
  Tests that explicitly assert fallback behavior are evidence for the production finding, not
  additional production bypasses.
- Stored `due_date_original`, `due_date_effective`, invoice `deduction_rate`, and client
  `vat_exempt_ceiling` snapshots when populated from the registry. Snapshotting itself is explicitly
  allowed by `docs/project/tax-rules-config.md`; TRC-037/TRC-039/TRC-041 concern ambiguous
  refresh/provenance semantics and current-value presentation, not the existence of snapshots.
- User-entered `advance_rate`, override amounts, and custom annual deadlines. These are per-client
  operational inputs, not official global constants.
- `VAT_TURNOVER_MISMATCH_TOLERANCE`, reminder windows, work-queue urgency windows, and the 80%
  warning threshold are product policies unless product documentation claims they are statutory.
- Decimal values such as `0.35`, `0.25`, or `25_000` used only as random demo probabilities or
  ordinary business amounts were rejected as numeric false positives.
- Generated OpenAPI examples and frontend generated types do not define tax behavior.

## Root causes

The findings reduce to five architectural causes:

1. **The canonical profile is not representable.** Missing persistence fields prevent use of the
   obligation resolver.
2. **Package access is optional at runtime.** Broad exception handlers turn canonical config
   failures into local behavior.
3. **Application enums model only an early subset of obligations.** Calendar and workflow objects
   cannot represent the full registry vocabulary.
4. **Tax-year inputs are optional or resolved at import time.** This creates cross-year leakage.
5. **Presentation and seed paths lack a config-derived DTO.** They duplicate values that the backend
   already knows.

## Recommended remediation order

This is sequencing guidance, not authorization to change tax behavior:

1. Remove broad exception fallbacks and make missing/unsupported registry data explicit.
2. Add the missing client-tax-profile fields and a single application adapter that constructs and
   validates `ClientTaxProfile`.
3. Make obligation generation consume `get_obligations`; extend application enums/models only for
   obligations the product intentionally supports.
4. Replace annual and periodic local deadline calculations with public registry results. Keep
   snapshots, but store the rule id/version/source alongside generated facts where needed.
5. Make every tax calculation require an explicit year; remove import-time resolution.
6. Expose config-derived display metadata to the frontend and seed builders.
7. Remove tax-significant model/repository defaults such as full deduction.
8. Decide explicitly whether the standalone HTML is retained as a prototype or removed from the
   workspace.

## Verification notes

Deep validation completed on 2026-07-27:

- 42 reviewed candidates have one inventory row and one matching detail section each.
- Dispositions after counter-review: 23 confirmed active, nine confirmed structural, six dormant
  hazards, three non-production findings, and one retracted candidate.
- All referenced repository paths were checked for existence (96 path references).
- Every current production caller of the default-argument findings was traced; TRC-011, TRC-013,
  TRC-023, and TRC-024 were downgraded to dormant hazards.
- TRC-027 was retracted because the 80% warning is product policy, not an official tax rule.
- TRC-034 was corrected: `requires_form_6111` is recommended in package notes but is not actually
  represented in `ObligationScope`.
- Tax-rules package verification passed: 31 tests.
- Focused backend verification for tax-calendar generation/bootstrap, annual tax engine, and client
  Excel import passed: 44 tests.

No production data was queried and no external tax-law reconciliation was performed. Findings
assert divergence from the repository's canonical `tax_rules_config`, not whether either value is
legally correct. “Confirmed” means confirmed against repository code and tests, not confirmed
against Israeli law or live production state.
