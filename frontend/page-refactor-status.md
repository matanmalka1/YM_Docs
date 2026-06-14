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
| UsersPage | `/settings/users` | `features/users/pages/UsersPage.tsx` | settings | 146 | medium | 3 | pending | 5+ modal states |
| ChargesPage | `/charges` | `features/charges/pages/ChargesPage.tsx` | list | 157 | medium | 4 | pending | notificationContext in page |
| ClientsPage | `/clients` | `features/clients/pages/ClientsPage.tsx` | list | 233 | high | 5 | pending | 3 modal states + URL param useEffect |
| BindersPage | `/binders` | `features/binders/pages/BindersPage.tsx` | list | 206 | high | 6 | pending | useBindersPageDialogs exists — validate completeness |
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
