## Scope

This file owns only:

- Read-only inventory of existing domain documentation.
- Risk classification for documentation ownership and drift.

This file must not contain:

- Canonical domain rules.
- Rewritten product behavior.
- Migration changes.
- Decisions that were not approved by the product owner.

Source of truth: reference

# Domain Docs Inventory

## Summary

- Total docs found: 22 backend app READMEs, 16 `backend/docs/` domain/spec files, 2 binder/domain-model satellite docs, and 3 frontend docs (`DESIGN.md`, `memory/MEMORY.md`, `components/ui/table/README.md`). The frontend has no per-feature domain docs despite 30 feature directories.
- Domains with likely clear documentation ownership: Clients, Businesses, Charges, Correspondence, Signature Requests, Reports, Users, Notifications, Permanent Documents, Search, Dashboard, Invoice, Authority Contact, Actions, Timeline.
- Domains with unclear or missing documentation ownership: Tasks, Reminders, Tax Calendar, Audit (no backend doc at all); Work Queue (doc exists but lives outside the app directory and has no app README).
- Highest-risk areas: VAT, Binders, and Advance Payments. Each has a legacy spec that appears to contradict the implemented README.

Observed pattern: `backend/app/<domain>/README.md` files are dated ("Last audited: …"), structured, and appear code-accurate. The `backend/docs/<domain>/` folders are mostly pre-implementation Hebrew domain specs or external-URL source maps that appear to have drifted from the code.

**2026-05-29 update:** All 18 `backend/app/<domain>/README.md` files have been converted to pointer-only files (one line: `See docs/domains/<domain>.md`). They are no longer authoritative. Canonical source of truth for all domains is `docs/domains/*.md` per `documentation-map.md:126-129`. Follow-up required: verify each canonical doc fully covers its legacy README before deletion.

## Inventory Rules

- This file is reference only.
- This file does not define domain behavior.
- Domain behavior remains owned by existing domain docs until a migration phase is approved.
- Do not use this file to implement product behavior directly.

## Domain Inventory

### Notifications

Status: Clear
Risk: Low

Existing docs:

- `backend/app/notification/README.md` — full product overview, domain model, service architecture, and API. Dated 2026-05-27 (schema v2 / Phase 2).

Likely source of truth:

- `backend/app/notification/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None observed.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/notification/README.md`
- Reference: —
- Archive: —

Notes:

- Most current and complete README. A good template for the eventual canonical format.

### Reminders

Status: Missing source of truth
Risk: Medium

Existing docs:

- None. The domain is described only as a gap in `backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md` (sections 3 and 6) and `backend/TODO.md`.

Likely source of truth:

- Unclear — no dedicated doc.

Duplicates:

- None.

Conflicts / drift:

- The domain-model summary states the executor `_execute()` is unimplemented (always marks reminders `FAILED`) and that reminders are not scheduled. Other domains' docs (for example annual reports) refer to creating/canceling reminders, which could lead an agent to assume reminders fire. This is `unclear` without code verification.

Missing docs:

- Yes — no `app/reminders/README.md`.

Recommended future structure:

- Canonical: new `app/reminders/README.md`
- Reference: domain-model gap notes
- Archive: —

Notes:

- Appears to be a partially-built domain. The actual (non-firing) state should be documented explicitly.

### Work Queue

Status: Clear (mislocated)
Risk: Low-Medium

Existing docs:

- `backend/docs/backend/domains/work-queue.md` — detailed and appears code-accurate (source types, build flow, scoping, urgency, schema). No app README.

Likely source of truth:

- `backend/docs/backend/domains/work-queue.md` — mandatory in practice.

Duplicates:

- None.

Conflicts / drift:

- None observed.

Missing docs:

- No content gap, but the doc sits under `docs/backend/domains/` instead of `app/work_queue/README.md`, which is inconsistent with every other domain.

Recommended future structure:

- Canonical: relocate to `app/work_queue/README.md` (later phase)
- Reference: —
- Archive: —

Notes:

- Only domain whose canonical doc lives outside its app directory.

### Tasks

Status: Missing source of truth
Risk: Medium

Existing docs:

- None. Referenced only inside `work-queue.md` and the domain-model summary.

Likely source of truth:

- Unclear.

Duplicates:

- None.

Conflicts / drift:

- None.

Missing docs:

- Yes — no `app/tasks/README.md`.

Recommended future structure:

- Canonical: new `app/tasks/README.md`
- Reference: —
- Archive: —

Notes:

- Tasks appear to be first-class (CRUD, source-linking, work-queue merge) but are undocumented.

### VAT

Status: Conflicting
Risk: High

Existing docs:

- `backend/app/vat_reports/README.md` — work-item lifecycle. Dated 2026-04-11.
- `backend/docs/vat_report/vat_reports_domain_summary.md` — Hebrew domain spec.
- `backend/docs/vat_report/vat_reports_deep_summary.md` — Hebrew domain spec.
- `backend/docs/vat_report/VAT_DEFINITIONS.md` — English domain definitions for modeling.
- `backend/docs/vat_report/source-map.md` — external government/legal URLs only.

Likely source of truth:

- `backend/app/vat_reports/README.md` — mandatory. The `vat_report/*` files appear historical/reference.

Duplicates:

- Three overlapping VAT domain-definition docs: `vat_reports_domain_summary.md`, `vat_reports_deep_summary.md`, `VAT_DEFINITIONS.md`.

Conflicts / drift:

- Status model mismatch. The legacy `vat_reports_domain_summary.md` (section 5) defines statuses `draft, submitted, paid, refund_pending, refund_approved, zero_report`. The implemented README defines `pending_materials, material_received, data_entry_in_progress, ready_for_review, filed` (plus `canceled`). The legacy doc describes a filing/payment model; the code implements an intake/data-entry model.

Missing docs:

- None (the live README is strong).

Recommended future structure:

- Canonical: `backend/app/vat_reports/README.md`
- Reference: `VAT_DEFINITIONS.md`, `source-map.md`
- Archive: `vat_reports_domain_summary.md`, `vat_reports_deep_summary.md`

Notes:

- Highest-confidence contradiction found. An agent reading the legacy summary would build the wrong status machine.

### Tax Calendar

Status: Missing source of truth
Risk: Medium-High

Existing docs:

- None. This is a central integration point (`tax_calendar_entry_id` FK on VAT, annual reports, and advance payments; `materialization_service.py`) documented only indirectly in the domain-model summary and other domains' READMEs.

Likely source of truth:

- Unclear.

Duplicates:

- None.

Conflicts / drift:

- Deadline-sync behavior is described only from the annual-reports side, where the annual-reports README already flags related bugs. Cross-domain calendar behavior is otherwise `unclear`.

Missing docs:

- Yes — no `app/tax_calendar/README.md`.

Recommended future structure:

- Canonical: new `app/tax_calendar/README.md`
- Reference: domain-model summary
- Archive: —

Notes:

- High structural importance with no direct documentation. Priority gap.

### Annual Reports

Status: Duplicated
Risk: Medium

Existing docs:

- `backend/app/annual_reports/README.md` — detailed; includes Known Limitations and Open Tasks. Dated 2026-04-21.
- `backend/docs/annual_reports/annual_reports_summary.md` — Hebrew domain summary.
- `backend/docs/annual_reports/report_history_financial_decision.md` — a product decision document (ADR-like) about financial data in the report-history table.
- `backend/docs/annual_reports/source-map.md` — external government/legal URLs.

Likely source of truth:

- `backend/app/annual_reports/README.md` — mandatory.

Duplicates:

- `annual_reports_summary.md` overlaps the README domain content at a higher level.

Conflicts / drift:

- No sharp contradiction; the summary is higher-level Hebrew rather than conflicting. `report_history_financial_decision.md` is a real decision that appears to belong in an ADR rather than loose under `docs/`.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/annual_reports/README.md`
- Reference: `source-map.md`
- Archive: `annual_reports_summary.md`; promote `report_history_financial_decision.md` to an ADR

Notes:

- The README already carries an Open Tasks list with unresolved HIGH items (2026 tax brackets and NI ceilings). Those are product/tax-law decisions, not documentation cleanup. ~~multi-business auto-populate~~ resolved as accepted-design in `docs/domains/annual-reports.md`.

### Binders

Status: Conflicting
Risk: High

Existing docs:

- `backend/app/binders/README.md` — thin (38 lines): two-field lifecycle (`location_status` / `capacity_status`) plus lifecycle endpoints.
- `backend/app/binders/binder_domain_definitions.md` — 330 lines: richer model (`BinderIntake`, `BinderIntakeMaterial`, grouped multi-binder handover events, structured reporting periods).
- `backend/docs/binder_lifecycle_refactor_spec.md` — 1162 lines: historical refactor task spec for the two-field migration, which now appears implemented.

Likely source of truth:

- `backend/app/binders/README.md` — mandatory for the lifecycle field model, but incomplete.
- `binder_domain_definitions.md` — unclear: aspirational versus implemented is not stated.

Duplicates:

- Three binder docs with three different scopes.

Conflicts / drift:

- The README describes only lifecycle fields and endpoints. `binder_domain_definitions.md` asserts the model "is not fully accurate" without `BinderIntake` / `BinderIntakeMaterial` / grouped handover. Whether intake, material rows, and grouped handover are actually built is not derivable from the docs and is `unclear` without a code check.
- `binder_lifecycle_refactor_spec.md` is a completed migration spec (its target two-field model now matches the README) but is not labeled as historical.

Missing docs:

- A single canonical binder doc covering intake, lifecycle, and handover.

Recommended future structure:

- Canonical: `backend/app/binders/README.md` (expanded)
- Reference: `binder_domain_definitions.md` (after confirming what is implemented)
- Archive: `binder_lifecycle_refactor_spec.md`

Notes:

- Highest ambiguity. Do not act on `binder_domain_definitions.md` without verifying against code.

### Charges

Status: Clear
Risk: Low

Existing docs:

- `backend/app/charge/README.md` — lifecycle, model, API, cross-domain notes. Dated 2026-03-22.

Likely source of truth:

- `backend/app/charge/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None observed.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/charge/README.md`
- Reference: —
- Archive: —

Notes:

- Cross-domain notes (work-queue, invoice, actions) appear accurate.

### Clients

Status: Migrated
Risk: Low

Existing docs:

- `docs/domains/clients.md` ★ — canonical source of truth. Last verified 2026-05-29.
- `backend/app/clients/README.md` — legacy pointer only; no longer authoritative.
- `backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md` — reference; cross-cutting verification of identity model.

Likely source of truth:

- `docs/domains/clients.md` — mandatory.

Duplicates:

- `backend/app/clients/README.md` is a legacy pointer per documentation-map.md:126-129.

Conflicts / drift:

- `DOMAIN_MODEL_REVIEW_SUMMARY.md` documents two open gaps now recorded in `clients.md` Known Issues: (1) soft-delete has no downstream lifecycle policy; (2) entity_type change does not reconcile existing workflow snapshots.

Missing docs:

- None.

Recommended future structure:

- Canonical: `docs/domains/clients.md`
- Reference: `DOMAIN_MODEL_REVIEW_SUMMARY.md`
- Archive: `backend/app/clients/README.md` (pointer only, delete after full verification)

Notes:

- —

### Legal Entities

Status: Migrated
Risk: Low

Existing docs:

- `docs/domains/legal-entities.md` ★ — canonical source of truth. Last verified 2026-06-10.

Likely source of truth:

- `docs/domains/legal-entities.md` — mandatory.

Duplicates:

- Historical clients docs mention the pre-extraction location for `LegalEntity`, `Person`, and `PersonLegalEntityLink`.

Conflicts / drift:

- None observed after Phase 1 extraction.

Missing docs:

- None.

Recommended future structure:

- Canonical: `docs/domains/legal-entities.md`
- Reference: `docs/architecture/migration-phases.md`
- Archive: —

Notes:

- Phase 1 extracted models and repositories only. No legal-entities router or service layer exists yet.

### Advance Payments

Status: Conflicting
Risk: High

Existing docs:

- `backend/app/advance_payments/README.md` — detailed services, repositories, and API.
- `backend/docs/advance_payments_spec.md` — Hebrew pre-development spec, self-labeled "מקור אמת לפני פיתוח" ("source of truth before development").

Likely source of truth:

- `backend/app/advance_payments/README.md` — mandatory (describes the actual model).

Duplicates:

- The spec overlaps the README scope.

Conflicts / drift:

- Status model mismatch. The spec proposes removing `overdue` from the enum and splitting into `payment_status` (`pending|partial|paid`) plus computed `timing_status` (`on_time|overdue`) and computed `paid_late`. The implemented README model is a single `AdvancePaymentStatus: pending, paid, partial, overdue`. The spec's redesign appears not implemented, yet the doc still labels itself a source of truth.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/advance_payments/README.md`
- Reference: —
- Archive: `advance_payments_spec.md` (or convert to an ADR if the redesign is still intended)

Notes:

- The "source of truth before development" label is risky now that the code diverges. Relabel or archive as a priority.

### Reports

Status: Clear
Risk: Low

Existing docs:

- `backend/app/reports/README.md` — management/operational reports and exports. Dated 2026-03-17.

Likely source of truth:

- `backend/app/reports/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None. Distinct from "Annual Reports"; naming proximity is the only risk.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/reports/README.md`
- Reference: —
- Archive: —

Notes:

- —

### Timeline / History / Audit

Status: Duplicated (concept spread across files)
Risk: Medium

Existing docs:

- `backend/app/timeline/README.md` — operational feed. Dated 2026-05-08.
- `backend/docs/history-vs-timeline.md` — conceptual Timeline-versus-AuditTrail boundary.
- `backend/docs/history-map.md` — frontend map of history/timeline/audit usages and naming recommendations.
- `backend/app/audit/` — no README.

Likely source of truth:

- Timeline: `backend/app/timeline/README.md` — mandatory.
- Audit: unclear (no backend doc); the concept is defined only in `history-vs-timeline.md`.

Duplicates:

- Timeline/history/audit concepts are described in three places (timeline README, `history-vs-timeline.md`, `history-map.md`).

Conflicts / drift:

- Not contradictory — the docs agree (do not merge timeline with audit). However the rules are scattered, and the `audit` domain itself is undocumented.

Missing docs:

- Yes — `app/audit/README.md`.

Recommended future structure:

- Canonical: `backend/app/timeline/README.md` plus new `app/audit/README.md`
- Reference: `history-vs-timeline.md` (or promote the "do not merge" rule to an ADR); keep `history-map.md` as frontend reference
- Archive: —

Notes:

- `history-map.md` is frontend-focused and already flags naming drift (`History` vs `AuditTrail` vs `StatusHistory`).

### Signature Requests

Status: Clear
Risk: Low

Existing docs:

- `backend/app/signature_requests/README.md` — lifecycle, public token flow, immutable audit. Dated 2026-03-22.

Likely source of truth:

- `backend/app/signature_requests/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None observed.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/signature_requests/README.md`
- Reference: —
- Archive: —

Notes:

- Tightly coupled to annual reports `PENDING_CLIENT` (runs inside its transaction); the annual-reports README already flags the "no business → silent skip" bug.

### Correspondence

Status: Clear
Risk: Low

Existing docs:

- `backend/app/correspondence/README.md` — per-client-record correspondence entries. Dated 2026-03-17.

Likely source of truth:

- `backend/app/correspondence/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None. The timeline README lists correspondence as not yet a timeline source, which is consistent.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/correspondence/README.md`
- Reference: —
- Archive: —

Notes:

- —

### Businesses

Status: Clear
Risk: Low

Existing docs:

- `backend/app/businesses/README.md` — business activities under a client. Dated 2026-04-13.

Likely source of truth:

- `backend/app/businesses/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None. The identity model (Business under LegalEntity) is consistent with clients and the domain-model summary.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/businesses/README.md`
- Reference: —
- Archive: —

Notes:

- —

### Users

Status: Clear
Risk: Low

Existing docs:

- `backend/app/users/README.md` — authentication, user lifecycle, RBAC, audit logs. Dated 2026-05-26; audit basis includes a passing test run.

Likely source of truth:

- `backend/app/users/README.md` — mandatory.

Duplicates:

- None.

Conflicts / drift:

- None observed.

Missing docs:

- None.

Recommended future structure:

- Canonical: `backend/app/users/README.md`
- Reference: —
- Archive: —

Notes:

- —

### Other Discovered Domains

These in-scope domains have a dated app README and appear to have clear documentation ownership at Low risk, unless noted:

- Permanent Documents — `backend/app/permanent_documents/README.md` (2026-03-22). Clear / Low.
- Search — `backend/app/search/README.md` (2026-03-22). Clear / Low.
- Dashboard — `backend/app/dashboard/README.md` (2026-03-17). Clear / Low.
- Invoice — `backend/app/invoice/README.md` (2026-03-17). Clear / Low. The domain-model summary flags that provider integration is incomplete.
- Authority Contact — `backend/app/authority_contact/README.md` (2026-04-13). Clear / Low.
- Actions — `backend/app/actions/README.md` (2026-04-23). Clear / Low. Cross-domain executable-action registry.
- Notes — no README. Missing source of truth / Low (small domain).
- Health, Infrastructure, Middleware — READMEs exist; these are infrastructure, not product domains.

## Do Not Touch Yet

The following must not be edited until a domain docs migration phase is approved:

- All `backend/app/*/README.md` (the live, highest-value docs — do not edit during inventory).
- `backend/docs/vat_report/*` (4 files).
- `backend/docs/annual_reports/*` (3 files).
- `backend/docs/advance_payments_spec.md`.
- `backend/docs/binder_lifecycle_refactor_spec.md`.
- `backend/app/binders/binder_domain_definitions.md`.
- `backend/docs/backend/domains/work-queue.md`.
- `backend/docs/history-vs-timeline.md` and `backend/docs/history-map.md`.
- `backend/docs/frontend_screen_spec.md`.
- `backend/docs/domain_model/*`.
- All Hebrew content in any file — do not translate or normalize.

## Open Questions

The following require product-owner or code-verification input:

1. Binders: Is the `binder_domain_definitions.md` model (`BinderIntake`, `BinderIntakeMaterial`, grouped handover) implemented, partially implemented, or aspirational? This determines conflict versus roadmap and needs a code check.
2. Advance Payments: Is the `payment_status` / `timing_status` redesign in `advance_payments_spec.md` still intended? If yes, capture it as an ADR plus backlog; if no, archive the spec.
3. VAT: Are the legacy status concepts (`refund_pending`, `refund_approved`, `zero_report`) dropped permanently or planned future work? The code currently shows none of them.
4. Reminders and Tax Calendar: Should the current (incomplete) state be documented now, or after the domains are finished?
5. Naming: Should the `history-map.md` convention (`AuditTrail` / `StatusHistory` / `Timeline` / explicit `ReportHistory`) be adopted for future docs and code?
6. Out of scope but flagged (architecture, not domain): `backend/ARCHITECTURE.md` overlaps `docs/architecture/backend.md`, and `backend/docs/api-contract-standard.md` overlaps `docs/architecture/api-contracts.md`. Should these be reduced to pointers to the canonical architecture layer?

## Proposed Next Phase

Proposal only — do not implement.

Phase 4 — Conflict Resolution and Source-of-Truth Labeling (no domain rewrites):

1. Add the mandatory scope header (`## Scope … Source of truth: mandatory | reference | historical`) to each existing domain doc as labeling only, with no content change — declaring app READMEs `mandatory` and legacy specs `historical` or `reference`.
2. Resolve the three high-risk contradictions (VAT statuses, Advance Payments statuses, Binders model) only after the Open Questions above are answered.
3. Fill the four missing-ownership gaps (Tasks, Reminders, Tax Calendar, Audit) with thin app READMEs that document the actual current state, including "not implemented" where true.
4. Relocate `work-queue.md` to `app/work_queue/README.md` for consistency.
5. Promote genuine decisions (`report_history_financial_decision.md`, the timeline-versus-audit rule) into the `docs/adr/` layer.

This proposed phase stays inventory-adjacent: label, fill gaps, and resolve confirmed contradictions. It does not rewrite or normalize domain content or Hebrew.
