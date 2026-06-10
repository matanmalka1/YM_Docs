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
| AdvancePaymentsPage | `/tax/advance-payments` | `features/advancedPayments/pages/AdvancePaymentsPage.tsx` | list | 295 | high | 8.1 | pending | 4 state vars + complex batch logic — was missing from original plan |
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
