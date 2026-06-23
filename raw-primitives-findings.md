# Raw Primitives TODO

Generated from grep searches against `frontend/src` on 2026-06-23.

This is an actionable TODO artifact only. It is not a frontend source-of-truth document.

## Scope

Included:

- Feature, layout, and shared frontend code under:
  - `frontend/src/features`
  - `frontend/src/components/layout`
  - `frontend/src/components/shared`
- Raw native element usage.
- Broader design-system reinvention patterns: card chrome, badges/chips, progress bars/dots, and clickable non-button patterns.

Excluded:

- Shared UI implementation internals under `frontend/src/components/ui`, where raw tags are usually expected.
- Static design-system demo HTML.
- `node_modules`, build output, and package lock files.

## Summary

Raw native primitives outside shared UI internals:

| Tag | Count |
| --- | ---: |
| `button` | 29 grep hits |
| `input` | 9 |
| `table` | 5 |
| `a` | 4 |
| `textarea` | 1 |
| `select` | 0 |
| labeled raw checkbox | 0 |

Note: the `button` count includes one comment-only grep hit in
`frontend/src/features/clients/components/edit/ClientEditForm.tsx:24`; actual raw `<button>` markup
is 28 hits.

Broader reinvention buckets:

| Pattern | Count |
| --- | ---: |
| Card/container chrome reinvention | 33 |
| Badge/chip/pill reinvention | 26 |
| Progress/dot primitive drift | 12 |
| Clickable non-button patterns | 6 |
| Semantic alert/banner box reinvention | 19 |
| Raw table substructure tags | 85 |
| Tab/segmented-control reinvention | 11 |
| Local overlay/popover chrome | 13 |
| Direct `Loader2 animate-spin` spinner use | 2 |
| Raw SVG shape markup | 6 |
| Inline styles or hardcoded color values | 23 |
| Local ARIA widget role implementation | 10 |
| Button primitive style signature reinvention | 4 |
| Badge primitive style signature reinvention | 17 |
| Card primitive style signature reinvention | 6 |
| Input/container field style signature reinvention | 14 |

## Raw Native Elements

Search:

```sh
rg -n "<(button|input|select|textarea|table|a)\b" \
  frontend/src/features frontend/src/components/layout frontend/src/components/shared \
  --glob '*.tsx'
```

TODOs:

- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:51:      <button`
- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:155:                <button`
- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:230:            <button type="button" onClick={handleClear} className="p-1 text-gray-400 hover:text-gray-600">`
- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:313:      <button`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:54:        <button`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:63:        <button`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:80:            <input`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:84:      <button`
- [x] `frontend/src/features/annualReports/components/statusTransition/TransitionTargetSelector.tsx:11:          <button`
- [x] `frontend/src/features/invoices/components/ChargeInvoiceSection.tsx:61:                <a`
- [x] `frontend/src/features/tasks/components/list/TaskListColumns.tsx:37:        <button`
- [x] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:39:    <table className="w-full text-xs">`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:238:              <button`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:250:          <button`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:258:          <button`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:133:                <a href={CLIENT_ROUTES.detail(c.id)} className="underline font-medium" target="_blank" rel="noreferrer">`
- [x] `frontend/src/features/notes/components/NotesCard.tsx:89:            <button`
- [x] `frontend/src/features/annualReports/components/shared/CreateReportModal.tsx:59:        <input type="hidden" {...register('client_id')} />`
- [x] `frontend/src/features/importExport/components/ImportExportModal.tsx:75:          <input`
- [x] `frontend/src/features/annualReports/components/shared/OverdueBanner.tsx:59:            <button`
- [x] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:28:          <table className="w-full text-sm">`
- [x] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:78:          <button`
- [x] `frontend/src/features/annualReports/components/shared/ClientAnnualReportsTab.tsx:94:            <button`
- [x] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:32:      <button`
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:162:                <button`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:54:        <table className="w-full border-collapse text-sm">`
- [x] `frontend/src/features/charges/components/list/ChargeBulkToolbar.tsx:32:            <textarea`
- [x] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:18:        <table className="w-full text-xs">`
- [x] `frontend/src/features/charges/components/form/ChargesCreateModal.tsx:145:        <input type="hidden" {...register('client_record_id')} />`
- [x] `frontend/src/features/clients/components/edit/ClientEditForm.tsx:24:  /** Exposed form id so a parent can submit via <button form="...">. */`
- [x] `frontend/src/features/clients/components/details/ClientStatusCard.tsx:26:  <button`
- [x] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:117:      <table className="w-full border-collapse text-right text-sm">`
- [x] `frontend/src/features/vatReports/components/detail/VatBreakdownCards.tsx:79:          <button`
- [x] `frontend/src/features/binders/components/sections/BinderIntakesSection.tsx:147:                            <button`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:134:            <button`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:161:        <button`
- [x] `frontend/src/features/timeline/components/TimelineCard.tsx:40:    <button`
- [x] `frontend/src/features/reports/components/AgingReportCards.tsx:38:        <button`
- [x] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:16:        <button`
- [x] `frontend/src/features/vatReports/components/form/VatWorkItemsCreateModal.tsx:139:              <input type="hidden" {...register('client_id')} />`
- [x] `frontend/src/features/vatReports/components/form/VatWorkItemsCreateModal.tsx:158:          <input type="hidden" {...register('period')} />`
- [x] `frontend/src/features/vatReports/components/form/VatFileModal.tsx:99:          <input type="hidden" {...register('submission_method')} />`
- [x] `frontend/src/features/clients/components/details/ClientInfoSection.tsx:27:        <a href={`tel:${client.phone}`} className="text-primary-600 hover:underline">`
- [x] `frontend/src/features/clients/components/details/ClientInfoSection.tsx:37:        <a href={`mailto:${client.email}`} className="text-primary-600 hover:underline">`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:94:              <button`
- [x] `frontend/src/features/documents/components/form/DocumentsUploadCard.tsx:220:        <input`
- [x] `frontend/src/features/documents/components/list/DocumentCard.tsx:73:    <button`
- [x] `frontend/src/features/documents/components/list/DocumentsDataCards.tsx:196:      <input ref={fileInputRef} type="file" aria-label="העלאת קובץ" className="hidden" onChange={handleFileChange} />`

## Disposition: Raw `<button>`s — Now Dedicated Primitives

Decision (2026-06-23): these raw `<button>`s are **not** action buttons, so were **not** forced into
`<Button>` (an action pill: `rounded-full`, label + optional `icon`). They are different interaction
patterns (selection state, navigation, disclosure) and each became its own primitive. Status below.

- **Tabs + SegmentedControl → unified `primitives/SegmentedControl.tsx`** (variants:
  `underline`/`tabBar`/`boxed`/`choice`/`vertical`/`switch`). One "single-select among options"
  primitive covers both tab bars and toggle switches.
  - **A11y:** the active item is exposed via `aria-current`; toggle adopters pass `aria-pressed`.
    It deliberately does **not** use `role="tab"` — the ARIA tabs keyboard pattern (roving
    tabindex / arrow-key nav / `aria-controls`→`tabpanel`) is not implemented, so claiming the
    role would over-promise. If full keyboard-tabs are wanted later, add that pattern then.
  - Adopted: ClientDetailsTabBar, VatWorkItemDetailPage, AnnualReportFullPanel (tab bars);
    ClientSidebar (group-by switch), TransitionTargetSelector (status choice),
    ClientAnnualReportsTab (year selector).
- **Chip → `primitives/Chip.tsx`** (`Chip` interactive toggle + `ChipLabel` display badge).
  Adopted: NotesCard (tag toggles + tag label), TimelineCommandBar (filter chips).
- **ClickableCard → `primitives/ActionSurface.tsx`.** Adopted: QuickActionsPanel, ClientStatusCard,
  OverdueBanner, AgingReportCards, SignatureRequestsDashboardPanel, TimelineCard (disclosure pill).
  - **Pick the element by intent:** `ActionSurfaceLink` (`to=`) for navigation — preserves
    cmd/ctrl-click, "open in new tab", and a real `href`; `ActionSurfaceButton` for in-page actions
    (open modal/drawer, toggle, select). Navigating via a button's `onClick` is a regression this
    split exists to prevent. AgingReportCards (whole-card nav) and ClientStatusCard tiles use the
    Link; their disabled tiles fall back to a non-interactive `ActionSurfaceButton disabled`.
  - **Authoring gotcha:** variant hover/active styles must use plain `hover:`/`active:`, never the
    `enabled:` pseudo — `enabled:`/`disabled:` apply only to `<button>`/form elements and silently
    no-op on `ActionSurfaceLink` (`<a>`), so the link would lose its hover. To suppress hover on a
    disabled button, use `disabled:hover:bg-transparent` (as `compact` does), not an `enabled:` guard.
- **CarouselDots → `primitives/CarouselDots.tsx`.** Adopted: SeasonInsightsCarousel slide dots.
  Prev/next arrows use `<Button shape="square">`.
- **DismissBackdrop → `primitives/DismissBackdrop.tsx`.** Adopted: ClientSidebar mobile
  click-outside backdrop.

Out of scope (stay raw, not a primitive):

- `frontend/src/features/documents/components/form/DocumentsUploadCard.tsx:161` — drag-and-drop zone (`role="button"`).

Migrated to `<Button>` (stale `[ ]` entries above — close them):
`TaskListColumns.tsx:37`, `VatBreakdownCards.tsx:79`, `NavbarMoreMenu.tsx:84`, `Navbar.tsx:54`/`:63`,
`BinderIntakesSection.tsx:147`.

## High-Signal Raw Native Findings

Raw tables:

- [x] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:39`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:54`
- [x] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:28`
- [x] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:117`
- [x] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:18`

Raw file inputs:

- [x] `frontend/src/features/importExport/components/ImportExportModal.tsx:75`
- [x] `frontend/src/features/documents/components/list/DocumentsDataCards.tsx:196`
- [x] `frontend/src/features/documents/components/form/DocumentsUploadCard.tsx:220`

Raw hidden inputs:

- [x] `frontend/src/features/annualReports/components/shared/CreateReportModal.tsx:59`
- [x] `frontend/src/features/charges/components/form/ChargesCreateModal.tsx:145`
- [x] `frontend/src/features/vatReports/components/form/VatWorkItemsCreateModal.tsx:139`
- [x] `frontend/src/features/vatReports/components/form/VatWorkItemsCreateModal.tsx:158`
- [x] `frontend/src/features/vatReports/components/form/VatFileModal.tsx:99`

Hidden form registration inputs remain raw intentionally. They are React Hook Form plumbing, not
visual controls, and wrapping them in a UI primitive would not centralize a reusable interaction or style.

Raw textarea:

- [x] `frontend/src/features/charges/components/list/ChargeBulkToolbar.tsx:32`

## Card Or Container Chrome Reinvention

Search pattern targeted `rounded-xl` / `rounded-2xl` / `rounded-3xl` plus border and white/gray/slate surface classes.

- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:142`
- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:179`
- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:199`
- [ ] `frontend/src/components/layout/NotificationBell.tsx:17`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:112`
- [ ] `frontend/src/components/layout/Navbar/Navbar.tsx:58`
- [ ] `frontend/src/components/layout/Navbar/Navbar.tsx:86`
- [ ] `frontend/src/features/signing/pages/SigningPage.tsx:6`
- [ ] `frontend/src/features/signing/components/SigningForm.tsx:96`
- [ ] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:14`
- [ ] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:28`
- [ ] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:51`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:94`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:95`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157`
- [ ] `frontend/src/features/dashboard/components/panels/OpenChargesCard.tsx:15`
- [ ] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:11`
- [ ] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:51`
- [ ] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:76`
- [ ] `frontend/src/features/annualReports/components/shared/ClientAnnualReportsTab.tsx:29`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:103`
- [ ] `frontend/src/features/auth/pages/ResetPasswordPage.tsx:95`
- [ ] `frontend/src/features/auth/pages/ResetPasswordPage.tsx:110`
- [ ] `frontend/src/features/auth/pages/ForgotPasswordPage.tsx:74`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:75`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:86`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:102`
- [ ] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:70`
- [ ] `frontend/src/features/reports/components/AgingReportCards.tsx:42`
- [ ] `frontend/src/features/timeline/components/TimelineEventItem.tsx:187`
- [ ] `frontend/src/features/documents/components/list/DocumentCard.tsx:38`
- [ ] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:91`
- [ ] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:113`

## Badge, Chip, Or Pill Reinvention

Search pattern targeted `rounded-full` with pill sizing, label text sizes, boldness, min-width counters, or tabular numbers.

- [x] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:70`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:47`
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76`
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:109`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:188`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:225`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:319`
- [x] `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:94`
- [x] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:36`
- [x] `frontend/src/features/search/components/SearchFiltersBar.tsx:41`
- [x] `frontend/src/features/search/components/DocumentResultsSection.tsx:95`
- [x] `frontend/src/features/tasks/components/list/TaskListColumns.tsx:72`
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:188`
- [ ] `frontend/src/features/notes/components/NotesCard.tsx:123`
- [x] `frontend/src/features/notifications/components/list/NotificationListItem.tsx:12`
- [ ] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:28`
- [ ] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:35`
- [x] `frontend/src/features/vatReports/components/shared/VatProgressBar.tsx:14`
- [x] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:98`
- [x] `frontend/src/features/timeline/components/TimelineCard.tsx:45`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:139`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:165`
- [x] `frontend/src/features/documents/components/detail/DocumentVersionsPanel.tsx:36`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientReviewStep.tsx:95`

## Progress Or Dot Primitive Drift

Search pattern targeted `rounded-full` progress tracks/bars and small status dots.

- [ ] `frontend/src/features/dashboard/components/panels/TaxInsightsRow.tsx:28`
- [ ] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`
- [ ] `frontend/src/features/annualReports/components/season/SeasonProgressBar.tsx:16`
- [ ] `frontend/src/features/signing/components/SigningForm.tsx:41`
- [ ] `frontend/src/features/signing/components/SigningForm.tsx:63`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:79`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:106`
- [ ] `frontend/src/features/annualReports/components/tax/DeductionsTab.tsx:50`
- [ ] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:50`
- [ ] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:52`
- [ ] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:66`
- [ ] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:68`

## Clickable Non-Button Patterns

Search pattern targeted `role="button"`, explicit `cursor-pointer`, and clickable non-button/non-link elements.

- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:262`
- [ ] `frontend/src/features/reports/components/AgingReportCards.tsx:42`
- [ ] `frontend/src/features/binders/components/sections/BinderIntakesSection.tsx:48`
- [ ] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:91`
- [ ] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:103`
- [ ] `frontend/src/features/documents/components/form/DocumentsUploadCard.tsx:161`

## Semantic Alert Or Banner Box Reinvention

Search pattern targeted local rounded boxes combining semantic border and background colors such as
`border-warning-* bg-warning-*`, `border-negative-* bg-negative-*`, `border-info-* bg-info-*`, and
similar red/yellow variants.

These may be candidates for `Alert`, `InlineState`, `StateCard`, or a purpose-built shared banner
component depending on context.

- [ ] `frontend/src/features/importExport/components/ImportExportModal.tsx:53`
- [ ] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:97`
- [ ] `frontend/src/features/signing/components/SigningForm.tsx:117`
- [ ] `frontend/src/features/vatReports/components/shared/VatFiledBanner.tsx:19`
- [ ] `frontend/src/features/annualReports/components/shared/CreateReportModalParts.tsx:45`
- [ ] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:82`
- [ ] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:128`
- [ ] `frontend/src/features/vatReports/components/detail/VatSummaryTab.tsx:16`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCalculatorInputs.tsx:38`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:47`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:51`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:55`
- [ ] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentEditableSections.tsx:46`
- [ ] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentDrawer.tsx:105`
- [ ] `frontend/src/features/clients/components/dialogs/DeletedClientDialog.tsx:59`
- [ ] `frontend/src/features/clients/components/details/DeleteClientModal.tsx:52`
- [ ] `frontend/src/features/annualReports/components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:39`
- [ ] `frontend/src/features/annualReports/components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:73`
- [ ] `frontend/src/features/notifications/components/form/SendNotificationModal.tsx:246`

## Raw Table Substructure Tags

Search pattern targeted `thead`, `tbody`, `tfoot`, `tr`, `th`, and `td` under feature/layout/shared
code. These are more detailed than the raw `<table>` findings above and show where table behavior is
locally hand-built.

Per-file counts:

- [ ] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:14`
- [ ] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:8`
- [ ] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:8`
- [ ] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:12`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:4`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:15`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:24`

High-signal locations:

- [ ] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:118`
- [ ] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:40`
- [ ] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:29`
- [ ] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:19`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:55`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:73`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:99`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:25`
- [ ] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:67`

## Tab Or Segmented-Control Reinvention

Search pattern targeted `role="tablist"`, `role="tab"`, `aria-selected`, and segmented-control
surface patterns such as `rounded-* bg-gray-100 p-1`.

These may be candidates for a shared tabs or segmented-control primitive if the interactions and
visual states are meant to stay consistent.

- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:259`
- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:153`
- [ ] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:76`
- [ ] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:81`
- [ ] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:82`
- [ ] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:92`
- [ ] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:97`
- [ ] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:98`
- [ ] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:11`
- [ ] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:19`
- [ ] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:20`

## Local Overlay Or Popover Chrome

Search pattern targeted local `fixed`/`absolute` overlay positioning combined with `z-*`,
`bg-black`, `bg-white`, border, rounded, or shadow chrome.

Some of these may be intentional layout shell pieces, but they bypass shared overlay/menu primitives
and should be reviewed before adding more local overlay behavior.

- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:55`
- [ ] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:71`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:112`
- [ ] `frontend/src/components/layout/Navbar/Navbar.tsx:43`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:305`
- [ ] `frontend/src/features/dashboard/components/panels/UpcomingDeadlinesPanel.tsx:45`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:47`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:146`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:147`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:148`
- [ ] `frontend/src/features/timeline/components/TimelineEventItem.tsx:175`
- [ ] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:30`

## Direct Loader Spinner Use

Search pattern targeted `Loader2` and `animate-spin`. Direct spinner markup is rare; most loading
states already route through `Button`, `Spinner`, or page/table state components.

Actual direct animated spinner usages:

- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:105`
- [ ] `frontend/src/features/workQueue/components/workQueueColumns.tsx:257`

Import-only hits from the same search:

- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:5`
- [ ] `frontend/src/features/workQueue/components/workQueueColumns.tsx:2`

## Raw SVG Shape Markup

Search pattern targeted raw SVG and shape tags. These are not Lucide icons; both hits are local
dashboard ring chart implementations.

- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:19`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:20`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:21`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:31`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:32`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:33`

## Inline Styles Or Hardcoded Color Values

Search pattern targeted `style=`, hex colors, and raw `rgb`/`rgba`/`hsl` values outside shared UI
internals and tests.

Some width styles are legitimate dynamic layout values, but several overlap with existing
`ProgressBar`, chart/token, or design-token concerns.

- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:248`
- [ ] `frontend/src/components/layout/PageHeader.tsx:64`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:138`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:139`
- [ ] `frontend/src/features/annualReports/components/season/SeasonProgressBar.tsx:25`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:32`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:40`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:50`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:109`
- [ ] `frontend/src/features/vatReports/components/detail/VatCategoryTable.tsx:83`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:44`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:52`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:62`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:76`
- [ ] `frontend/src/features/dashboard/components/panels/RecentActivityPanel.tsx:53`
- [ ] `frontend/src/features/dashboard/components/panels/TaxInsightsRow.tsx:31`
- [ ] `frontend/src/features/annualReports/constants/financialConstants.ts:48`
- [ ] `frontend/src/features/annualReports/constants/financialConstants.ts:49`
- [ ] `frontend/src/features/annualReports/constants/financialConstants.ts:50`
- [ ] `frontend/src/features/annualReports/constants/financialConstants.ts:51`
- [ ] `frontend/src/features/reports/components/AgingReportCards.tsx:65`
- [ ] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:69`
- [ ] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:53`

## Local ARIA Widget Role Implementation

Search pattern targeted local `combobox`, `listbox`, `option`, `menu`, `menuitem`, `status`, and
`alert` role usage plus `aria-haspopup` / `aria-activedescendant`.

These are not automatically wrong, but they indicate locally implemented widgets that may deserve a
shared primitive or accessibility review.

- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:219`
- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:222`
- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:246`
- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:257`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:33`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:103`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:113`
- [ ] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:136`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:103`
- [ ] `frontend/src/features/annualReports/components/shared/OverdueBanner.tsx:26`

## Reinvented Primitive Style Signatures

These searches compare non-primitive callers against the actual class signatures used by core
primitives such as `Button`, `Badge`, `Card`, and `Input`.

They intentionally overlap with earlier buckets, but are stricter: the matches are based on class
shape rather than broad UI category.

### Button-Like Style Signature

Count: 4

- [ ] `frontend/src/components/layout/Navbar/Navbar.tsx:58`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21`
- [ ] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`

### Badge-Like Style Signature

Count: 17

- [ ] `frontend/src/components/layout/Navbar/Navbar.tsx:47`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:109`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:188`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:225`
- [ ] `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:94`
- [ ] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`
- [ ] `frontend/src/features/auth/pages/LoginPage.tsx:188`
- [ ] `frontend/src/features/search/components/DocumentResultsSection.tsx:95`
- [ ] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:98`
- [ ] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:28`
- [ ] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:35`
- [ ] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:139`
- [ ] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:165`
- [ ] `frontend/src/features/timeline/components/TimelineCard.tsx:45`
- [ ] `frontend/src/features/notifications/components/list/NotificationListItem.tsx:12`

### Card-Like Style Signature

Count: 6

- [ ] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:51`
- [ ] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:94`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:95`
- [ ] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157`
- [ ] `frontend/src/features/dashboard/components/panels/OpenChargesCard.tsx:15`
- [ ] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:51`

### Input Or Field-Container Style Signature

Count: 14

- [ ] `frontend/src/components/shared/client/ClientSearchInput.tsx:247`
- [ ] `frontend/src/features/notes/components/NotesCard.tsx:60`
- [ ] `frontend/src/features/notes/components/NotesCard.tsx:121`
- [ ] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:91`
- [ ] `frontend/src/features/annualReports/components/tax/TaxCreditsPanel.tsx:24`
- [ ] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentDrawer.tsx:110`
- [ ] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:35`
- [ ] `frontend/src/features/clients/components/createClientModal/CreateClientTaxStep.tsx:72`
- [ ] `frontend/src/features/clients/components/details/ClientStatusCard.tsx:103`
- [ ] `frontend/src/features/clients/components/details/ClientStatusCard.tsx:135`
- [ ] `frontend/src/features/clients/components/edit/ClientEditFormSections.tsx:32`
- [ ] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:40`
- [ ] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:50`
- [ ] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:156`

## Strict Primitive Checks

No findings outside shared UI internals:

```text
raw <select>: 0
raw labeled checkbox: 0
```

Expected shared UI internal hits were seen under `frontend/src/components/ui`, including primitive implementations for inputs, buttons, tables, select, textarea, and shared table/action components. Those are intentionally excluded from this report.
