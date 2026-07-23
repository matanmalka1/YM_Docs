## Scope

This file owns only:

- The product roadmap and decision record for upgrading the advance-payments screens, derived from the `tax-advances-system.html` design exploration (repo root, generated demo).

This file must not contain:

- Canonical current-state advance-payments behavior (that lives in `docs/domains/advance-payments.md`).
- Architecture rules or endpoint contracts (link to `docs/architecture/*`).
- Implementation details for work already merged.

Source of truth: tracking only — not source of truth for current behavior.

# Advance Payments — Product Roadmap

Decision record dated 2026-07-23, from a gap review of the demo (`tax-advances-system.html`) against the real screens (`frontend/src/features/advancedPayments/`). The demo is a design exploration, not a spec: every capability below was individually accepted, rejected, or deferred against the office's actual workflow.

Lifecycle facts verified in code (no spec needed): mid-year client join is already handled — `generate_annual_schedule` skips periods whose due date has passed; `advance_rate` and frequency seed from the legal entity at creation.

## Standing decisions (confirmed with product owner, 2026-07-23)

| Topic | Decision |
|---|---|
| Payment model | Single payment per period stays. Add one field: `payment_reference` (free-text, nullable). No multi-payment ledger. |
| Interest & linkage (§190) | Indication only — an "interest likely accruing" badge/note on overdue unpaid records. No local interest computation, no interest table. The Tax Authority's שע"מ figure is authoritative. |
| Case owner (אחראי תיק) | Real need, deferred. Client-level field affecting all modules — separate cross-cutting project, backlog. |
| Calculation bases (§181ב excess expenses / fixed-amount notice) | Only a handful of such clients — `override_amount` suffices; no basis selector. |
| Withheld-at-source credit (ניכוי במקור) | Real correctness gap — add a `withheld_amount` credit field (Phase 1). |
| Client payment reminders | Yes — via the existing notifications module (manual, template-first), not a new mechanism (Phase 2). |
| Rate change after 2216 approval | Bulk "from period X onward" rate-update action (Phase 1), not per-period manual edits. |
| Screen scope | Every feature applies to **both** the org list and the client-tab. New fields/badges are shared components (cheap). List tooling (chips/sort/CSV/selection) is page-level wiring and the tab renders cards, not tables — adapt per view at implementation time. |
| Reports module | New fields (`withheld_amount`, `payment_reference`) are added as columns to `AdvancePaymentReportView` in the same phase they are built (Phase 1). |
| Permissions | Payment reminder: SECRETARY allowed (follows the notifications module's existing router policy). Audit-trail viewing: all roles. Edit / delete / batch mark-paid / bulk rate update: ADVISOR only (unchanged). |
| Batch mark-paid on partial rows | Partial payments are included: the action tops up `paid_amount` to `expected_amount` (client settled the difference). Only fully-paid rows are skipped. |
| Reminder recipient | Same recipient-resolution logic as all other notifications — no advances-specific contact field. |
| National-insurance advances (מקדמות ב"ל) | Real office need, consciously deferred — a new domain with its own spec, separate project after the income-tax phases. Backlog. |
| Delivery path | Spec first (this document), then Linear issues, then implementation in phases. |

## Phase 0 — frontend-only quick wins

No backend changes. All data already served.

1. **Days-overdue in status badge.** "בפיגור N ימים" computed from `due_date_effective ?? due_date`; today only the date turns red.
2. **Free-text client search.** Wire the existing `client_search` overview param into the filter panel (name / ת.ז / office number). Keep the exact client picker; per `feedback_client_filter_semantics`, these are distinct filters (fuzzy vs exact) and must not merge.
3. **Unsaved-changes guard** on the detail page — `beforeunload` (refresh / tab close) only. Full in-app navigation blocking requires `useBlocker`, which needs a data router; the app mounts a declarative `<BrowserRouter>`, so router migration is a backlog item. Modal/drawer close guarding already exists (`ui/overlays/useUnsavedChangesGuard`).

## Phase 1 — small backend + frontend

1. **`payment_reference` field.** Nullable string on `AdvancePayment`; PATCH/create payloads, detail form input, overview row + CSV export column.
2. **Batch mark-paid.** Row selection on batch tables + bulk endpoint: date, method, optional reference prefix; sets `paid_amount = expected_amount`. Returns done/skipped counts (skip already-paid). Mirrors the existing bulk refresh-turnover shape (`BulkRefreshTurnoverResponse`).
3. **Export via the reports domain** (decided 2026-07-23, superseding both the client-side CSV idea and an overview-level export): `GET /reports/advance-payments/export?format=excel|pdf` exports the existing collections report (year/month) in **Excel and PDF**, mirroring the aging-report export exactly (same exporters, same route shape, same two-button UI). Exports live in the reports domain; the org list screen itself does not export.
4. **Sorting via server `sort_by` + `order` params** on the overview endpoint (client_name / expected_amount / paid_amount / delta, stable id tiebreaker). UI decided 2026-07-23: two selects in the FilterPanel — the app's existing sort pattern (clients screen) — NOT clickable table headers; the DataTable primitive stays untouched. Rows are server-paginated, so client-side sorting was never an option.
5. **"Overdue" as a status-filter option.** Decided 2026-07-23 (replaces the earlier quick-chips idea): overdue is filtered like a status, via the existing status select. Requires a real `timing_status` filter param on the overview endpoint — no client-side interim. "Unpaid" needs no new UI (= the existing pending option); "due this week" was dropped (advance due dates are fixed office-wide and move only on authority postponements).
6. **VAT turnover mismatch flag.** When `turnover_source` is `vat_*` but the stored `turnover_amount` was later edited away from the VAT figure (or when a VAT report exists and amounts differ beyond a tolerance), surface a mismatch state: warning on the detail screen with the difference, ⚠ marker in the list, and a quick-filter. Backend computes the flag (frontend must not re-derive; cross-repo contract).
7. **Delete reason.** Required free-text reason on delete, persisted to the audit trail (audit module exists).
8. **Audit trail on the detail page.** Read surface for the entity audit trail (change log per save: field, old → new, who, when). Depends on the generic entity-audit surface; no access-log tab (demo-only).
9. **Interest indication (per standing decision).** Overdue + unpaid ⇒ small warning note "ריבית והצמדה נצברות לפי סעיף 190 — הסכום הרשמי בשע\"מ". No amounts.
10. **`withheld_amount` credit field (ניכוי במקור).** Nullable amount on `AdvancePayment`, subtracted from the calculated amount: `expected = max(0, turnover × rate − withheld)`; `override_amount` still wins when set. Detail form input + calc-breakdown line; overview/CSV column. Without it, `expected_amount` is simply wrong for withheld clients.
11. **Bulk rate update "from period X onward".** Small endpoint: new `advance_rate` + starting period → updates all the client's future unpaid periods in one action, audited. This is the execution arm of an approved 2216 (see Phase 2.5) and of any authority rate notice.
12. **Office-wide annual schedule generation.** One action "צור לוחות שנה לכל הלקוחות": runs the existing per-client generator over all active clients with a configured frequency, skips existing periods, returns per-client created/skipped summary. Kills the December click-marathon (today the modal is per-client only).
13. **Frequency-change cleanup in the generator.** When regenerating after a legal-entity frequency change, soft-delete unpaid future rows left over from the old frequency (audited); paid/partial rows are kept. Verified gap: `exists_for_period` matches on the exact `YYYY-MM` key, so stale rows from the old cadence otherwise linger alongside the new ones.

## Phase 2 — UX depth (after Phase 1 ships)

1. **Period prev/next navigation** on the detail page: siblings = same client, same year; picker + arrows.
2. **Annual context card** on the detail page, fed by the existing `clientAdvancePaymentsKPI` endpoint (year totals: expected, paid, balance).
3. **VAT-mismatch filter** on the org list — once the Phase 1 mismatch flag exists (server-computed filter, matching the overdue-filter approach).
4. **Payment reminder to client.** Button on the advances list/detail invoking the existing notifications flow (manual, template-first, per the notifications spec) with an advance-payment trigger + template. Integration spec only — no new delivery mechanism, no batch send in v1.
5. **2216 rate-reduction tracking — decided (2026-07-23): task-based, no bespoke workflow.** Submission happens in שע"מ outside the system; the system only tracks. Model as a regular task in the tasks module ("הגשת 2216 ללקוח X", due date, follow-up); on approval the advisor updates `advance_rate` on the payment (existing field, audited). Revisit only if 2216 volume grows enough to justify dedicated infrastructure. Rate application uses the Phase 1.11 bulk "from period X onward" action.

## Rejected (with reasons — do not re-propose without new evidence)

| Demo capability | Why rejected |
|---|---|
| ₪ KPI amounts on the org list (overdue ₪, collected ₪, progress) | Product owner 2026-07-23: the accountant works by counts of clients needing action, not aggregate sums — money totals are not a decision signal here. Existing count cards stay. |
| Bank reconciliation (CSV import + matching) | Clients pay the Tax Authority directly; the office's bank account never sees these payments. Nothing to reconcile against. |
| Payment vouchers + bank-transfer file | Payment is digital (authority site / שע"מ / direct debit). If a need appears, send payment details via the notifications module instead. |
| Full §190 interest computation | Standing decision: indication only. Local computation drifts from the authority's official figure and adds a rate-table maintenance burden. |
| Multi-payment ledger | Standing decision: single payment + reference suffices for the real workflow. |
| Calculation-basis selector (§181ב excess / fixed amount) | Only a handful of affected clients; `override_amount` covers them. A selector + two more formulas is infrastructure for edge cases. |
| Offline queue + IndexedDB drafts | Heavy machinery; office works online. |
| Optimistic-lock conflict dialog (version/ETag, diff, force) | Backend has no record versioning; small office, low collision odds. |
| Three-role model (viewer/editor/admin) | Current model is `isAdvisor`; role expansion is a product-wide decision, not an advances feature. |
| Filed/locked (דווח לרשות המסים) state | Filing happens in שע"מ, outside the system, so this is only a manual checkbox; not worth a state machine now. Backlog at most. |
| Alerts bell with advance alerts | Overlaps the notifications spec (single automatic trigger policy). |
| Access log tab, role switcher, simulated concurrency | Demo theater. |

## Backlog (real, not now)

- Case owner (אחראי תיק) as a client-level field: owner column, "mine" filter, batch assign — cross-module project.
- National-insurance advances (מקדמות ביטוח לאומי) — new domain, own spec; `tax_rules_config` already models NI obligations.
- Router migration to `createBrowserRouter` (data router) — unlocks `useBlocker` for full in-app unsaved-changes blocking.
- Keyboard shortcuts on the detail page (Ctrl+S save, Esc back, Alt+arrows period nav).
- Print stylesheet for list/detail.

## Follow-up

- Open Linear issues per phase once this spec is approved.
- Update `docs/domains/advance-payments.md` only as phases actually merge (current-state doc must not lead the code).
