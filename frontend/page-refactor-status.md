## Scope

This file owns only:
- Refactor status tracking for all frontend pages.

Update this file after every step. Never start a new page without updating the previous page's status first.

Source of truth: reference

---

# Frontend Page Refactor Status

Status values:
- **compliant** — already matches standard, no changes needed
- **in-progress** — refactor started, not yet reviewed
- **done** — refactor complete, review passed, verification passed
- **skipped** — deferred with reason

---

| Page | Route | File | Type | Lines | Risk | Step | Status | Notes |
|------|-------|------|------|-------|------|------|--------|-------|
| TaxDashboardPage | `/tax/dashboard` | `features/taxDashboard/pages/TaxDashboardPage.tsx` | dashboard | 22 | low | 1 | compliant | 22 lines, uses useTaxDashboard(), no direct API calls, TaxSubmissionStats is presentational. activeFilter is UI-only local state — correct. |
| AnnualReportsPage | `/tax/reports` | `features/annualReports/pages/AnnualReportsPage.tsx` | list | 88 | medium | 2 | compliant | Uses useAnnualReportsPage() only, no direct API/query calls, all components presentational. taxYearLabel inline derive is display-only — acceptable. |
| AnnualReportDetailPage | `/tax/reports/:reportId` | `features/annualReports/pages/AnnualReportDetailPage.tsx` | detail | 14 | low | 2 | compliant | 14 lines, routing params + Navigate guard only, delegates to AnnualReportFullPanel feature widget. |
| VatComplianceReportView | `/reports/vat-compliance` | `features/reports/components/VatComplianceReportView.tsx` | list | ~100 | low | 2.5 | pending | Component not in pages/ — structural move only |
| AgingReportView | `/reports/aging` | `features/reports/components/AgingReportView.tsx` | list | ~80 | low | 2.5 | pending | Component not in pages/ — structural move only |
| AnnualReportStatusView | `/reports/annual-status` | `features/reports/components/AnnualReportStatusView.tsx` | list | ~25 | low | 2.5 | pending | Component not in pages/ — structural move only |
| AdvancePaymentReportView | `/reports/advance-payments` | `features/reports/components/AdvancePaymentReportView.tsx` | list | ~55 | low | 2.5 | pending | Component not in pages/ — structural move only |
| UsersPage | `/settings/users` | `features/users/pages/UsersPage.tsx` | settings | 80 | medium | 3 (Wave 1) | done | Wave 1. Grouped contract (`status`/`headerProps`/`filters`/`table`/`modals`/`permissions`; no stats). Columns + `currentUserId`/`useAuthStore` + `pendingToggle` toggle dialog + all modal prop objects moved into `useUsersPage`; page is pure slot composition (no useState/useMemo/useAuthStore/buildUserColumns/inline handlers). Additive `filters.resetFilters` (EC-5, no new UI — reuses existing FilterPanel reset). Header buttons + non-advisor branch stay page-composed (sanctioned). Automated verification passed; no unit tests / no renderHook infra. See page-convergence-changes.md. |
| ChargesPage | `/charges` | `features/charges/pages/ChargesPage.tsx` | list | 80 | medium | 4 (Wave 2) | done | Wave 2 + contract checkpoint. Grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`/`drawers`/`modals`/`permissions`). Columns + `allIds` + selected-charge/create/notification state + both URL-param effects (charge_id deep-link, create=1 strip) + `useSearchParamFilters` moved into `useChargesPage`; page is pure slot composition (no useState/useEffect/useMemo/useSearchParamFilters/buildChargeColumns/inline handlers). `filters.resetFilters` converged from inline clear. `table.selection` nested sub-group for bulk actions; empty state stays component-owned in ChargesTableBlock (checkpoint decision). Advisor header button stays page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no charges unit tests / no renderHook infra. See page-convergence-changes.md (Wave 2 + CONTRACT CHECKPOINT). |
| ClientsPage | `/clients` | `features/clients/pages/ClientsPage.tsx` | list | 233 | high | 5 | pending | 3 modal states + URL param useEffect |
| BindersPage | `/binders` | `features/binders/pages/BindersPage.tsx` | list | 75 | high | 6 (Wave 0A+0B) | done | Reference page. 0A: grouped contract + columns/dialogs into `useBindersPage`. 0B: receive-drawer ownership, pre-wired `drawers.detail` / `drawers.receive` / `modals.dialogsProps` moved into the hook — page is now pure slot composition (75 lines, no useState/handlers/copy). Automated verification + Atlas smoke passed. See page-convergence-changes.md. |
| WorkQueuePage | `/work-queue` | `features/workQueue/pages/WorkQueuePage.tsx` | list | 160 | medium | 7 | pending | All state in hooks — verify |
| VatWorkItemsPage | `/tax/vat` | `features/vatReports/pages/VatWorkItemsPage.tsx` | list | 142 | medium | 8 | pending | showCreateModal in page, URL params for create/client_id/period |
| AdvancePaymentsPage | `/tax/advance-payments` | `features/advancedPayments/pages/AdvancePaymentsPage.tsx` | list | 102 | high | 8.1 | in-progress | Page is now a composition shell backed by `useAdvancePaymentsPage`; batch querying, grouping, table rows, navigation, URL validation, and drawer mutation behavior were extracted. Automated verification passed. Browser smoke testing remains pending. |
| NotificationsPage | `/notifications` | `features/notifications/pages/NotificationsPage.tsx` | list | 335 | high | 8.2 | pending | URL state violation: clientQuery shadow state |
| SearchPage | `/search` | `features/search/pages/SearchPage.tsx` | list | 121 | low | 8.3 | pending | Verify queryDraft — fix only if it duplicates URL state |
| TaxCalendarSettingsPage | `/settings/tax-calendar` | `features/taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx` | settings | 402 | high | 8.5 | pending | Largest page — nested CRUD modals |
| ClientDetailsPage | `/clients/:clientId` | `features/clients/pages/ClientDetailsPage.tsx` | detail | 193 | medium | 9 | pending | Phase 1 only — no routing change |
| BusinessDetailsPage | `/clients/:clientId/businesses/:businessId` | `features/businesses/pages/BusinessDetailsPage.tsx` | detail | 60 | low | 9.5 | pending | Check-only — likely compliant |
| VatWorkItemDetailPage | `/tax/vat/:id` | `features/vatReports/pages/VatWorkItemDetailPage.tsx` | detail | 133 | low | 9.5 | pending | Check-only — verify tab state |
| TaxCalendarGroupsPage | `/tax/calendar` | `features/taxCalendar/pages/TaxCalendarGroupsPage.tsx` | list | 91 | low | 9.5 | pending | Check-only — debounce already correct |
| DashboardPage | `/` | `features/dashboard/pages/DashboardPage.tsx` | dashboard | 207 | high | 10 | pending | 7+ modals, cross-domain coupling — last |
| LoginPage | `/login` | `features/auth/pages/LoginPage.tsx` | public | 199 | low | skipped | skipped | Auth/public flow — separate phase |
| ForgotPasswordPage | `/forgot-password` | `features/auth/pages/ForgotPasswordPage.tsx` | public | 96 | low | skipped | skipped | Auth/public flow — separate phase |
| ResetPasswordPage | `/reset-password` | `features/auth/pages/ResetPasswordPage.tsx` | public | 147 | low | skipped | skipped | Auth/public flow — separate phase |
| SigningPage | `/sign/:token` | `features/signing/pages/SigningPage.tsx` | public | 73 | low | skipped | skipped | Public client-facing flow — separate phase |

## Wave 0A — Binders (reference page, contract-only)

`BindersPage` is the canonical reference. Wave 0A established the grouped page-hook
contract on this page only; it did **not** move modal/drawer ownership (Wave 0B).

Completed:

- `useBindersPage` now returns the grouped contract: `status` (`isLoading` /
  `isFetching` / `error: string | null` / `loadingMessage`), `headerProps`,
  `stats`, `filters` (incl. `resetFilters`), and `table`
  (`data` / `columns` / `pagination` / `emptyState`).
- Column construction (`buildBindersColumns`) moved out of the page into the hook;
  the page no longer imports `buildBindersColumns`.
- `useBindersPageDialogs` is now composed inside `useBindersPage` (necessary so the
  column action callbacks can reference dialog handlers). Dialog/selection **state
  ownership is unchanged** — still owned by `useBindersPageDialogs` /
  `useBinderSelection`; only the call site relocated, grouped under `modals` /
  `drawers` for the page to compose JSX from.
- `table.emptyState` is an `EmptyStateConfig` (`isEmpty`, `isFiltered`, `title`,
  `message`, `action`); loading/error/empty derivation moved into the hook.
- `BindersPage` is now slot composition: `const page = useBindersPage(...)` reading
  `page.status` / `page.headerProps` / `page.stats` / `page.filters` /
  `page.table.*`. Page no longer derives loading/error/empty copy.
- Receive drawer (`receiveOpen`) and its action button stay page-owned per the
  Wave 0A boundary; the hook receives `onOpenReceive` so it still owns the
  empty-state action copy.

Side-effects handled (behavior-preserving):

- Wrapped `pageItems` in `useMemo` — moving the columns memo into the hook surfaced
  an unstable-ref lint warning the old page-level memo had masked.

Verification completed:

- `npm run typecheck` (zero binder errors), `npm run lint` (binder files clean),
  `npm run arch:check` (no violations), `npm run build` (green).
- No binder unit tests exist — none run.

Required before changing status to `done`:

- Browser smoke-test `/binders`: loading/error/empty states, stats filter chips,
  filters bar + reset, table pagination + row click → detail drawer, receive
  drawer, and the delete / handover / bulk dialogs.

Deferred to Wave 0B:

- Move dialog wiring, receive flow, and detail-drawer selected-entity state fully
  behind the hook; fold `useBindersPageDialogs`' separate dialog states into the
  `modals` / `drawers` groups.

## Step 8.1 — Advance Payments

Completed:

- Moved URL filters, permissions, batch grouping, workflow counters, overlay state, navigation, and update mutation behavior out of `AdvancePaymentsPage`.
- Replaced unsafe URL casts with explicit validation for client record ID, payment status, and reporting frequency.
- Split the batch implementation into the summary row, server-state hook, table content, table row, and table constants.
- Replaced browser-level `_self` navigation with React Router navigation.
- Uses one authoritative server-paginated overview query per displayed due-date batch.
- Backend HTTP coverage verifies that filtering by one `due_date` returns both monthly and bimonthly payments sharing that deadline, with the correct total.
- Recalculates merged `collection_rate` from merged paid and expected totals.

Verification completed:

- Frontend lint, typecheck, architecture check, full Vitest suite, and production build.
- Focused advance-payment grouping tests, including invalid/null aggregate values and merged collection rate.
- Backend `tests/advance_payments/api/test_advance_overview_filters.py`.

Required before changing status to `done`:

- Browser smoke-test `/tax/advance-payments` at desktop and narrow widths, including opening a batch, pagination, row navigation, filters, and the update drawer. This remains pending because local dev-server permission was not granted during the refactor session.

Non-blocking follow-up:

- `AdvancePaymentDrawer.tsx` remains 433 lines and still owns substantial form state, validation, and VAT prefill behavior. Split it only as a dedicated drawer refactor with focused form tests.
- `useAdvancePaymentsPage` is 137 lines. Keep it as the page controller for now; split filters, batch groups, or drawer actions only if additional responsibilities are added.
- The production build still reports the existing large-chunk warning. Address bundle splitting separately from this page refactor.

## Annual Report Panels

Completed:

- Extracted tax calculation, income/expense, annex form, and status-transition behavior from presentation components into feature hooks.
- Typed income payloads end to end and validate income/expense enum values before constructing API payloads.
- Tax calculation error state now includes both calculation and report-detail query failures.
- Annex form reset behavior depends on stable `react-hook-form` methods instead of the complete form object.

Remaining:

- Browser-check the annual-report detail financial, annex, tax-calculation, and status-transition states before treating the widget-level visual verification as complete.
