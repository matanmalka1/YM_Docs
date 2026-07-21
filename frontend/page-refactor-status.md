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
| AnnualReportsPage | `/tax/reports` | `features/annualReports/pages/AnnualReportsPage.tsx` | list | 75 | medium | 2 (Wave 5) | done | Wave 5 (section-loading variant — NO PageStateGuard; amends plan's stale "switch to PageStateGuard" line, since filters must stay visible during load). Un-nested `season.*` → grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`/`banner`/`modals`). `taxYearLabel` derivation + `onSelect` report→id wrap + no-summary `StateCard` emptyState config moved into hook; page branches on `stats.summary` presence (slot placement). Header action button stays page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 5). |
| AnnualReportDetailPage | `/tax/reports/:reportId` | `features/annualReports/pages/AnnualReportDetailPage.tsx` | detail | 14 | low | 2 | compliant | 14 lines, routing params + Navigate guard only, delegates to AnnualReportFullPanel feature widget. |
| VatComplianceReportView | `/reports/vat-compliance` | `features/reports/pages/VatComplianceReportView.tsx` | report | ~100 | low | 2.5 (Wave 7) | done | Wave 7 (report archetype, NOT management-list — convention fix only). Moved `components/`→`pages/` (100% rename, 0 content change — inline tables live in-file). Hook `useVatComplianceReport` intentionally NOT renamed (folder is the fix). No table swap (multi-table). See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 7). |
| AgingReportView | `/reports/aging` | `features/reports/pages/AgingReportView.tsx` | report | ~80 | low | 2.5 (Wave 7) | done | Wave 7. Moved `components/`→`pages/`; sibling imports `./AgingReportHeader`/`./AgingReportTable`→`../components/…`. Hook `useAgingReport` NOT renamed. No table swap (custom `AgingReportTable` + `PaginationCard`). Dead unused `embedded?` prop left in place (smallest change). |
| AnnualReportStatusView | `/reports/annual-status` | `features/reports/pages/AnnualReportStatusView.tsx` | report | ~25 | low | 2.5 (Wave 7) | done | Wave 7. Moved `components/`→`pages/`; `./AnnualReportStatusTable`→`../components/…`. Hook `useAnnualReportStatusReport` NOT renamed. No pagination/table swap. Dead unused `taxYear?` prop left in place (smallest change). |
| AdvancePaymentReportView | `/reports/advance-payments` | `features/reports/pages/AdvancePaymentReportView.tsx` | report | ~55 | low | 2.5 (Wave 7) | done | Wave 7. Moved `components/`→`pages/`; `./AdvancePaymentReportTable`→`../components/…`. Hook `useAdvancePaymentReport` NOT renamed. No pagination/table swap. |
| UsersPage | `/settings/users` | `features/users/pages/UsersPage.tsx` | settings | 80 | medium | 3 (Wave 1) | done | Wave 1. Grouped contract (`status`/`headerProps`/`filters`/`table`/`modals`/`permissions`; no stats). Columns + `currentUserId`/`useAuthStore` + `pendingToggle` toggle dialog + all modal prop objects moved into `useUsersPage`; page is pure slot composition (no useState/useMemo/useAuthStore/buildUserColumns/inline handlers). Additive `filters.resetFilters` (EC-5, no new UI — reuses existing FilterPanel reset). Header buttons + non-advisor branch stay page-composed (sanctioned). Automated verification passed; no unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md). |
| ChargesPage | `/charges` | `features/charges/pages/ChargesPage.tsx` | list | 80 | medium | 4 (Wave 2) | done | Wave 2 + contract checkpoint. Grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`/`drawers`/`modals`/`permissions`). Columns + `allIds` + selected-charge/create/notification state + both URL-param effects (charge_id deep-link, create=1 strip) + `useSearchParamFilters` moved into `useChargesPage`; page is pure slot composition (no useState/useEffect/useMemo/useSearchParamFilters/buildChargeColumns/inline handlers). `filters.resetFilters` converged from inline clear. `table.selection` nested sub-group for bulk actions; empty state stays component-owned in ChargesTableBlock (checkpoint decision). Advisor header button stays page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no charges unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 2 + CONTRACT CHECKPOINT). |
| ClientsPage | `/clients` | `features/clients/pages/ClientsPage.tsx` | list | 88 | high | 5 (Wave 3) | done | Wave 3. Grouped contract (`status`/`isEmptyState`/`headerProps`/`stats`/`filters`/`table`/`drawers`/`modals`/`permissions`). Columns + edit-drawer state + 2nd `useClientQuery` + 3 modal states + `create=1` URL effect + row-click navigation + `useNavigate` moved into `useClientsPage`; page is pure slot composition (no useState/useEffect/useNavigate/useSearchParams/buildClientColumns/useClientQuery/empty-state derivation/inline handlers). Two sanctioned new components: `ClientsStatsSection` (replaces inline 3× StatsCard grid) + `ClientEditDrawer` (DetailDrawer+ClientEditForm+ModalFormActions). Raw `PaginatedDataTable` → hook supplies `table.emptyState` (rich empty state preserved); no `table.selection`. Header buttons + view-only Alert stay page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no clients unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 3). |
| BindersPage | `/binders` | `features/binders/pages/BindersPage.tsx` | list | 75 | high | 6 (Wave 0A+0B) | done | Reference page. 0A: grouped contract + columns/dialogs into `useBindersPage`. 0B: receive-drawer ownership, pre-wired `drawers.detail` / `drawers.receive` / `modals.dialogsProps` moved into the hook — page is now pure slot composition (75 lines, no useState/handlers/copy). Automated verification + Atlas smoke passed. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md). |
| WorkQueuePage | `/work-queue` | `features/workQueue/pages/WorkQueuePage.tsx` | list | 70 | medium | 7 (Wave 4) | done | Wave 4 (standard-validator — already exposed `isFetching`). Grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`/`modals`). Second-actions hook `useWorkQueueActions` composed INTO `useWorkQueuePage` (Binders precedent; state ownership unchanged) → page consumes ONE hook. Columns + `showLinkedTasks`/`showWarnings` + error-toast `useEffect` moved into the hook; page is pure slot composition (no 2nd hook/useEffect/useMemo/buildWorkQueueColumns/renderEmpty). `renderEmpty` → `EmptyStateConfig` (verified rendering equivalence — DataTable renders identical StateCard, same suppress-on-refetch gating; no renderEmpty passthrough needed). `stats` uses `isFetching`+`summaryError=error` (EC-5). Header button stays page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no workQueue unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 4). |
| TasksPage | `/tasks` | `features/tasks/pages/TasksPage.tsx` | list | 79 | medium | task-alignment | done | Replaced the custom operational list with `PaginatedDataTable`, retained URL-backed filters, and added retryable list errors. `TaskModal` now follows the shared RHF+Zod modal pattern with dirty-dismiss protection, field errors, and specific-user assignment. Client tasks now use the shared table, allow standalone client-scoped creation, and restrict bulk selection to open tasks. Verification: typecheck, task-feature lint, architecture check, Prettier, and production build pass. Browser flow verification pending: in-app browser unavailable and the local app requires authenticated task data. |
| VatWorkItemsPage | `/tax/vat` | `features/vatReports/pages/VatWorkItemsPage.tsx` | list | 67 | medium | 8 (Wave 5) | done | Wave 5 (section-loading variant — NO PageStateGuard; grouped-cards composite owns list loading/error/empty). Single-hook fold: `useVatWorkItemGroups` composed INTO `useVatWorkItemsPage` (Wave-5 checkpoint resolved → option a). Grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`/`modals`/`permissions`). Columns + filter→param mapping + `showCreateModal` + `create=1` strip effect + `client_id`/`period` initial values + `useNavigate` moved into hook; page is pure slot composition (no useState/useEffect/useMemo/useNavigate/useSearchParams/build*Columns/2nd hook). `stats.visible` gate moved into hook. `useVatWorkItemGroups` now exposes `isFetching`. Header button stays page-composed (sanctioned). Verification: typecheck/lint/arch/build/test (42 pass) all green; no unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 5). |
| AdvancePaymentsPage | `/tax/advance-payments` | `features/advancedPayments/pages/AdvancePaymentsPage.tsx` | list | 102 | high | 8.1 | in-progress | Page is now a composition shell backed by `useAdvancePaymentsPage`; batch querying, grouping, table rows, navigation, URL validation, and drawer mutation behavior were extracted. P0 domain follow-up: payment status is server-owned, the edit form no longer validates status from a local money preview, and the current-month counter comes from the backend batch contract. Typecheck, lint, formatting, architecture checks, and the full frontend test suite pass. Browser smoke testing remains pending. |
| NotificationsPage | `/notifications` | `features/notifications/pages/NotificationsPage.tsx` | list | 60 | high | 8.2 (Wave 6) | done | Wave 6. Created `useNotificationsPage` from scratch (no hook existed); grouped contract (`status`/`headerProps`/`permissions`/`filters`/`table`/`drawers`/`modals`). Decision A → extracted `buildNotificationColumns` (`components/table/NotificationsColumns.tsx`, owns getTriggerLabel/getDomainLabel). Decision B → extracted `NotificationDetailDrawer` (mirrors ClientEditDrawer; advisor footer gated via hook-bound `onSend`). Query + `useSearchParamFilters` + columns + detail/send state moved into the hook; page = guard + slots, advisor header button page-composed (sanctioned). Keeps `PageStateGuard` + generic error string (no getErrorMessage — smallest change). **Stale flag cleared: no clientQuery shadow state — client filter is URL-only (getParam + picker writes URL).** Verification: typecheck/lint/arch/build/test (47 pass) all green; no notifications page unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 6). |
| SearchPage | `/search` | `features/search/pages/SearchPage.tsx` | list | 17 | low | 8.3 + MAT-94 | done | MAT-94 (global search phase 1) rebuilt the page as **results-only**: one text input, no filters, no selection. One-hook composition shell (section-loading variant — NO PageStateGuard; the toolbar stays visible while results load). Grouped contract (`status`/`headerProps`/`toolbar`/`results`); page is pure slot composition. `useSearchPage` owns URL state (`search`/`type`/`page`/`page_size` only), the two queries (`/search` for client resolution + grouped match previews, `/search/items` for one type's paginated expansion, enabled only while a chip is active), the debounced draft, reset/focus, and all result-state derivation. **Not a table page** — the page is two labelled sections side by side (see `docs/domains/search.md`): **לקוחות תואמים** (`SearchClientMatches` — rows navigate to `/clients/:id`; never automatically, whatever the match count, D1) and **רשומות תואמות** (`SearchMatchesSection` — chip row with exact per-type totals + grouped previews of ≤5 rows, or the active type's full paginated list; rows are `SearchMatchRow`, each carrying its owning client's name + office number, click follows the row's `href`). `page` pages whichever list the user is in (spec §7.3): the clients list with no `type` active, the expansion with one — the clients section is then pinned to page 1 with no pager; changing/clearing `type` resets `page` in one atomic `setFilters` write. **MAT-63/64/87 machinery deleted as deliberately closed debt, not regressions** (spec §2: the selection contract and the filters no longer exist to be violated): `SearchFiltersBar` (+ its FilterPanel sanctioned-exception status), `SearchSelectedClient`, `SearchItemFeed`, `utils/searchSelection.ts`, the four enum URL guards, all `client_record_id` selection handling, and the advanced-panel open/close state (MAT-65). What survives of MAT-64 is the one-write URL cleanup mechanism: `SEARCH_DROPPED_FILTER_KEYS` now strips `client_record_id`, the four enum filter params, and `id_number`/`binder_number` from old deep links, and the only guarded param left is `type` (`isSearchMatchType`). Tests: `utils/searchUrlValues.test.ts` (dropped-keys list + type guard), `utils/searchMatches.test.ts` (chip derivation + §7.3 page rule), `components/SearchMatchesSection.test.tsx`, `components/SearchResultsSection.test.tsx`, `components/SearchToolbar.test.tsx`, `api/search.api.test.ts` (SSR-markup + pure fns; no jsdom/RTL in the stack). |
| TaxCalendarSettingsPage | `/settings/tax-calendar` | `features/taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx` | settings | 402 | high | 8.5 | pending | Largest page — nested CRUD modals |
| ClientDetailsPage | `/clients/:clientId` | `features/clients/pages/ClientDetailsPage.tsx` | detail | 193 | medium | 9 | pending | Phase 1 only — no routing change |
| BusinessDetailsPage | `/clients/:clientId/businesses/:businessId` | `features/businesses/pages/BusinessDetailsPage.tsx` | detail | 60 | low | 9.5 | pending | Check-only — likely compliant |
| VatWorkItemDetailPage | `/tax/vat/:id` | `features/vatReports/pages/VatWorkItemDetailPage.tsx` | detail | 84 | low | 9.5 | done | Section-loading variant (custom skeleton + inline error Alert; NO PageStateGuard — header lives inside loaded content, converting would change behavior). Extracted `useVatWorkItemDetailPage` (composes the `useVatWorkItemPage` query hook): tab URL-state + `isTabKey` guard, `isFilingPending`, error→string conversion, income/expense badge counts, `tabs` config, `headerProps` (title/breadcrumbs data), `filedBanner` props (or null). Page is now slot composition — no `useState`/`useSearchParams`/derived counts/tabs/banner-guard. Header `actions` = `<VatWorkItemHeaderActions>` stays page-composed (sanctioned, needs `workItem`); tab-content switch stays page JSX (slot fill). Tab state verified: URL `tab` param, guarded parse, `summary` clears param. Routing guard unchanged. Verification: tsc clean, eslint clean (changed files), arch:check clean. |
| TaxCalendarGroupsPage | `/tax/calendar` | `features/taxCalendar/pages/TaxCalendarGroupsPage.tsx` | list | 47 | low | 9.5 (Wave 6) | done | Wave 6 (section-loading variant — NO PageStateGuard; `TaxCalendarGroupsContent` owns loading/error/empty). Created `useTaxCalendarGroupsPage` from scratch (no hook existed); grouped contract (`status`/`headerProps`/`stats`/`filters`/`table`). Query + `useSearchParamFilters` + 350ms `useDebounce` + param mapping + `?? EMPTY_SUMMARY` default moved into the hook; page = bare `<div dir="rtl">` + slot components, header `size="lg"` preserved. Verification: typecheck/lint/arch/build/test (47 pass) all green; no taxCalendar page unit tests / no renderHook infra. See [frontend-page-convergence-log.md](../archive/frontend-page-convergence-log.md) (Wave 6). |
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
