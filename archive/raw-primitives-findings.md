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
| MonoValue reinvention (raw font-mono / direct tone classes) | 22 |
| DefinitionList reinvention (raw dl/dt/dd) | 3 |
| Divider reinvention candidates (border-t separators) | 30 |

Re-grep 2026-06-23: raw native elements now down to 6 hits — all intentional (5 hidden RHF
`<input type="hidden">` + 1 comment in `ClientEditForm.tsx:24`). Action-button / table / file-input /
textarea migrations from the original sweep are complete.

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

- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:142`
- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:179`
- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:199`
- [x] `frontend/src/components/layout/NotificationBell.tsx:17`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:112`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:58`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:86`
- [x] `frontend/src/features/signing/pages/SigningPage.tsx:6`
- [x] `frontend/src/features/signing/components/SigningForm.tsx:96`
- [x] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:14`
- [x] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:28`
- [x] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:51`
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:94`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:95`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157`
- [x] `frontend/src/features/dashboard/components/panels/OpenChargesCard.tsx:15`
- [x] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:11`
- [x] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:51`
- [x] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:76`
- [x] `frontend/src/features/annualReports/components/shared/ClientAnnualReportsTab.tsx:29`
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:103`
- [x] `frontend/src/features/auth/pages/ResetPasswordPage.tsx:95`
- [x] `frontend/src/features/auth/pages/ResetPasswordPage.tsx:110`
- [x] `frontend/src/features/auth/pages/ForgotPasswordPage.tsx:74`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:75`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:86`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:102`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:70`
- [x] `frontend/src/features/reports/components/AgingReportCards.tsx:42`
- [x] `frontend/src/features/timeline/components/TimelineEventItem.tsx:187`
- [x] `frontend/src/features/documents/components/list/DocumentCard.tsx:38`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:91`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:113`

Disposition: all candidates have now been reviewed. The migrated reusable surfaces use `Card` above.
The remaining entries are intentionally specialized: navigation/actions and overlay/menu chrome,
semantic alert/state boxes (covered by the alert group), form/field containers, timeline event content,
or existing `ActionSurface` interactions. `SeasonSummaryWidget.tsx` no longer exists; its entries were stale.

## Badge, Chip, Or Pill Reinvention

Search pattern targeted `rounded-full` with pill sizing, label text sizes, boldness, min-width counters, or tabular numbers.

- [x] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:70`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:47`
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76`
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:109`
- [x] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:188`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:225`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:319`
- [x] `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:94`
- [x] `frontend/src/features/tasks/components/list/TasksListSummary.tsx:36`
- [x] `frontend/src/features/search/components/SearchFiltersBar.tsx:41`
- [x] `frontend/src/features/search/components/DocumentResultsSection.tsx:95`
- [x] `frontend/src/features/tasks/components/list/TaskListColumns.tsx:72`
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:188`
- [x] `frontend/src/features/notes/components/NotesCard.tsx:123`
- [x] `frontend/src/features/notifications/components/list/NotificationListItem.tsx:12`
- [x] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:28`
- [x] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:35`
- [x] `frontend/src/features/vatReports/components/shared/VatProgressBar.tsx:14`
- [x] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:98`
- [x] `frontend/src/features/timeline/components/TimelineCard.tsx:45`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:139`
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:165`
- [x] `frontend/src/features/documents/components/detail/DocumentVersionsPanel.tsx:36`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientReviewStep.tsx:95`

Disposition: Login feature labels and the VAT assignee are now rendered with `Badge`; NotesCard was
already using `ChipLabel`. The VAT binder remains a real `Link` so normal navigation semantics are
preserved, and the dashboard onboarding item is a CTA link rather than a badge. The three
`SeasonSummaryWidget` entries are stale because that component no longer exists.

## Progress Or Dot Primitive Drift

Search pattern targeted `rounded-full` progress tracks/bars and small status dots.

- [x] `frontend/src/features/dashboard/components/panels/TaxInsightsRow.tsx:28`
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142`
- [x] `frontend/src/features/annualReports/components/season/SeasonProgressBar.tsx:16`
- [x] `frontend/src/features/signing/components/SigningForm.tsx:41`
- [x] `frontend/src/features/signing/components/SigningForm.tsx:63`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:79`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:106`
- [x] `frontend/src/features/annualReports/components/tax/DeductionsTab.tsx:50`
- [x] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:50`
- [x] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:52`
- [x] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:66`
- [x] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:68`

## Clickable Non-Button Patterns

Search pattern targeted `role="button"`, explicit `cursor-pointer`, and clickable non-button/non-link elements.

- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:262`
- [x] `frontend/src/features/reports/components/AgingReportCards.tsx:42`
- [x] `frontend/src/features/binders/components/sections/BinderIntakesSection.tsx:48`
- [x] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:91`
- [x] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:103`
- [x] `frontend/src/features/documents/components/form/DocumentsUploadCard.tsx:161`

Disposition: `ClientSearchInput` keeps its `role="option"` elements because the input owns keyboard
navigation through the ARIA combobox/listbox pattern. `AgingReportCards` already uses
`ActionSurfaceLink`. The VAT status now uses a real `Link` around its existing `Badge`, preserving
the badge design while exposing a navigable `href`. The handover row itself is not interactive; only
its labelled checkbox is, so its redundant pointer cursor was removed. `DocumentsUploadCard` remains
a purpose-built drag-and-drop file target with keyboard activation rather than being forced into an
action-button primitive.

## Semantic Alert Or Banner Box Reinvention

Search pattern targeted local rounded boxes combining semantic border and background colors such as
`border-warning-* bg-warning-*`, `border-negative-* bg-negative-*`, `border-info-* bg-info-*`, and
similar red/yellow variants.

These may be candidates for `Alert`, `InlineState`, `StateCard`, or a purpose-built shared banner
component depending on context.

- [x] `frontend/src/features/importExport/components/ImportExportModal.tsx:53`
- [x] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:97`
- [x] `frontend/src/features/signing/components/SigningForm.tsx:117`
- [x] `frontend/src/features/vatReports/components/shared/VatFiledBanner.tsx:19`
- [x] `frontend/src/features/annualReports/components/shared/CreateReportModalParts.tsx:45`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:82`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:128`
- [x] `frontend/src/features/vatReports/components/detail/VatSummaryTab.tsx:16`
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculatorInputs.tsx:38`
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:47`
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:51`
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:55`
- [x] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentEditableSections.tsx:46`
- [x] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentDrawer.tsx:105`
- [x] `frontend/src/features/clients/components/dialogs/DeletedClientDialog.tsx:59`
- [x] `frontend/src/features/clients/components/details/DeleteClientModal.tsx:52`
- [x] `frontend/src/features/annualReports/components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:39`
- [x] `frontend/src/features/annualReports/components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:73`
- [x] `frontend/src/features/notifications/components/form/SendNotificationModal.tsx:246`

Disposition: simple status and warning messages now use `Alert` without local class overrides:
ImportExportModal, SigningForm, DeletedClientDialog, SendNotificationModal, and the existing alert
uses in advance-payments and annual-report VAT auto-population. The remaining candidates are
intentional specialized surfaces: ClientDetailsPage combines a count badge with a navigation link;
VatFiledBanner is a persistent filed-state summary with structured metadata; TaxPreview, VatSummaryTab,
and TaxCalculationPanel render financial values or metric cards; TaxCalculatorInputs and the deleted-client
identity panel include editable controls or recovery actions; DeleteClientModal retains its emphasized
irreversible-action explanation; and the VAT auto-populate outer panel contains metrics and result detail.

## Raw Table Substructure Tags

Search pattern targeted `thead`, `tbody`, `tfoot`, `tr`, `th`, and `td` under feature/layout/shared
code. These are more detailed than the raw `<table>` findings above and show where table behavior is
locally hand-built.

Per-file counts:

- [x] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:14`
- [x] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:8`
- [x] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:8`
- [x] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:12`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:4`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:15`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:24`

High-signal locations:

- [x] `frontend/src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx:118`
- [x] `frontend/src/features/annualReports/components/annex/AnnexDataTable.tsx:40`
- [x] `frontend/src/features/annualReports/components/shared/ClientYearComparisonModal.tsx:29`
- [x] `frontend/src/features/annualReports/components/tax/TaxBracketsTable.tsx:19`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:55`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:73`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:99`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceRow.tsx:25`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:67`

Disposition: TaxCalendarGroupsTable, AnnexDataTable, ClientYearComparisonModal, TaxBracketsTable,
and VatInvoiceTable already use `PaginatedDataTable` or `DataTable`. `VatInvoiceRow.tsx` no longer
exists. VatInvoiceTable's custom totals row and `VatInvoiceEditRow` deliberately return raw
`<tr>/<td>` markup through `DataTable.renderFooter` and `DataTable.renderEditRow`; those cells must
remain table children to preserve valid table semantics and column alignment.

## Tab Or Segmented-Control Reinvention

Search pattern targeted `role="tablist"`, `role="tab"`, `aria-selected`, and segmented-control
surface patterns such as `rounded-* bg-gray-100 p-1`.

These may be candidates for a shared tabs or segmented-control primitive if the interactions and
visual states are meant to stay consistent.

- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:259`
- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:153`
- [x] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:76`
- [x] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:81`
- [x] `frontend/src/features/vatReports/pages/VatWorkItemDetailPage.tsx:82`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:92`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:97`
- [x] `frontend/src/features/annualReports/components/panel/AnnualReportFullPanel.tsx:98`
- [x] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:11`
- [x] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:19`
- [x] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:20`

Disposition: ClientSidebar, VatWorkItemDetailPage, AnnualReportFullPanel, and ClientDetailsTabBar
already use `SegmentedControl` variants. ClientSearchInput is an ARIA combobox/listbox, not a tab or
segmented control; its `role="option"` implementation is reviewed with the local ARIA widget group.

## Local Overlay Or Popover Chrome

Search pattern targeted local `fixed`/`absolute` overlay positioning combined with `z-*`,
`bg-black`, `bg-white`, border, rounded, or shadow chrome.

Some of these may be intentional layout shell pieces, but they bypass shared overlay/menu primitives
and should be reviewed before adding more local overlay behavior.

- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:55`
- [x] `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:71`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:112`
- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:43`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:305`
- [x] `frontend/src/features/dashboard/components/panels/UpcomingDeadlinesPanel.tsx:45`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:47`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:146`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:147`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:148`
- [x] `frontend/src/features/timeline/components/TimelineEventItem.tsx:175`
- [x] `frontend/src/features/clients/components/details/ClientDetailsTabBar.tsx:30`

Disposition: ClientSidebar is the responsive application shell, with `DismissBackdrop` and a focus trap
for its mobile drawer. NavbarMoreMenu is a navigation-specific, keyboard-managed menu using
`useDismissibleLayer`; it has no second consumer to justify a generic menu primitive. UpcomingDeadlinesPanel,
TimelineEventItem, and LoginPage entries are structural or decorative layout, not overlays. Navbar and
ClientDetailsTabBar entries are stale; `SeasonSummaryWidget.tsx` no longer exists.

## Direct Loader Spinner Use

Search pattern targeted `Loader2` and `animate-spin`. Direct spinner markup is rare; most loading
states already route through `Button`, `Spinner`, or page/table state components.

Actual direct animated spinner usages:

- [x] `frontend/src/features/auth/pages/LoginPage.tsx:105`
- [x] `frontend/src/features/workQueue/components/workQueueColumns.tsx:257`

Import-only hits from the same search:

- [x] `frontend/src/features/auth/pages/LoginPage.tsx:5`
- [x] `frontend/src/features/workQueue/components/workQueueColumns.tsx:2`

Disposition: both direct animated spinners now use `Spinner`; the corresponding Loader2 imports were removed.

## Raw SVG Shape Markup

Search pattern targeted raw SVG and shape tags. These are not Lucide icons; both hits are local
dashboard ring chart implementations.

- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:19`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:20`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:21`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:31`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:32`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:33`

Disposition: SeasonSummaryWidget is stale. SeasonInsightsCarousel's SVG circles are a data-driven donut
chart, not an icon; preserving the SVG is the appropriate implementation.

## Inline Styles Or Hardcoded Color Values

Search pattern targeted `style=`, hex colors, and raw `rgb`/`rgba`/`hsl` values outside shared UI
internals and tests.

Some width styles are legitimate dynamic layout values, but several overlap with existing
`ProgressBar`, chart/token, or design-token concerns.

- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:248`
- [x] `frontend/src/components/layout/PageHeader.tsx:64`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:138`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:139`
- [x] `frontend/src/features/annualReports/components/season/SeasonProgressBar.tsx:25`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:32`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:40`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:50`
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:109`
- [x] `frontend/src/features/vatReports/components/detail/VatCategoryTable.tsx:83`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:44`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:52`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:62`
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:76`
- [x] `frontend/src/features/dashboard/components/panels/RecentActivityPanel.tsx:53`
- [x] `frontend/src/features/dashboard/components/panels/TaxInsightsRow.tsx:31`
- [x] `frontend/src/features/annualReports/constants/financialConstants.ts:48`
- [x] `frontend/src/features/annualReports/constants/financialConstants.ts:49`
- [x] `frontend/src/features/annualReports/constants/financialConstants.ts:50`
- [x] `frontend/src/features/annualReports/constants/financialConstants.ts:51`
- [x] `frontend/src/features/reports/components/AgingReportCards.tsx:65`
- [x] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:69`
- [x] `frontend/src/features/annualReports/components/panel/ReadinessCheckPanel.tsx:53`

Disposition: ClientSearchInput positioning and PageHeader animation delay are runtime geometry.
LoginPage's dot-grid texture and SeasonInsightsCarousel's SVG/chart presentation require inline values.
SeasonProgressBar, AgingReportCards, AnnualPLSummary, and ReadinessCheckPanel use data-driven widths;
VatCategoryTable's inset shadow remains a Tailwind arbitrary value. RecentActivityPanel now uses `pt-1`
instead of fixed inline padding. Financial chart lines now use semantic CSS variables instead of hex
literals. SeasonSummaryWidget and TaxInsightsRow no longer exist.

## Local ARIA Widget Role Implementation

Search pattern targeted local `combobox`, `listbox`, `option`, `menu`, `menuitem`, `status`, and
`alert` role usage plus `aria-haspopup` / `aria-activedescendant`.

These are not automatically wrong, but they indicate locally implemented widgets that may deserve a
shared primitive or accessibility review.

- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:219`
- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:222`
- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:246`
- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:257`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:33`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:103`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:113`
- [x] `frontend/src/components/layout/Navbar/NavbarMoreMenu.tsx:136`
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:103`
- [x] `frontend/src/features/annualReports/components/shared/OverdueBanner.tsx:26`

Disposition (2026-06-24): **review-only bucket — all correct, no reinvention.** Each is a properly
implemented ARIA widget, not a hand-rolled approximation of an existing primitive:

- **ClientSearchInput** — full combobox/listbox pattern: `role="combobox"` on the `<Input>` owns
  keyboard nav (`aria-expanded`/`aria-controls`/`aria-activedescendant`), portal `role="listbox"`,
  `role="option"` rows with `aria-selected`. Already dispositioned in the clickable + tab buckets.
- **NavbarMoreMenu** — `role="menu"`/`menuitem` + `aria-haspopup="menu"`/`aria-expanded`, with a real
  roving-focus keyboard implementation (Arrow/Home/End/Escape) via `useDismissibleLayer`. Single
  consumer; no second caller to justify extracting a generic menu primitive (overlay-chrome bucket).
- **LoginPage:103** — `role="status"` + `aria-live="polite"` loading region wrapping `Spinner`; correct
  live-region usage, not a widget.
- **OverdueBanner:26** — `role="alert"` + `aria-live` on a structured multi-row negative banner
  (header + list + expander); correct semantic role, richer than a generic `Alert`.

No accessibility changes needed.

## Reinvented Primitive Style Signatures

These searches compare non-primitive callers against the actual class signatures used by core
primitives such as `Button`, `Badge`, `Card`, and `Input`.

They intentionally overlap with earlier buckets, but are stricter: the matches are based on class
shape rather than broad UI category.

> **Disposition (2026-06-24): stale re-scan — all closed.** This section was generated against a
> snapshot that **predates** the Button/Badge/Card/Chip/ActionSurface migrations done in the earlier
> buckets, so its line numbers point at pre-migration markup. Every cited line was spot-verified
> against the current source: each is now either a real primitive (`Button` / `Badge` / `Card` /
> `Chip` / `ActionSurface`) or an intentional divergence already dispositioned above (a real `<Link>`
> for navigation, or inline text that is not a pill). Several `SeasonSummaryWidget` and
> `DashboardLayout:94/109` entries are stale because those lines no longer exist. No class-shape
> reinvention remains; nothing to convert.

### Button-Like Style Signature

Count: 4

- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:58` — now `<Button shape="square">`.
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76` — now `<Card variant="soft">` / `<Badge>`.
- [x] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21` — intentional CTA `<Link>` (real `href` navigation), not a button.
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142` — now `<Badge>` + `<Button>`.

### Badge-Like Style Signature

Count: 17

- [x] `frontend/src/components/layout/Navbar/Navbar.tsx:47` — now `<Badge variant="neutral">`.
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:76` — now `<Badge>` via `DashboardSectionHeader`.
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:109` — stale (line no longer exists).
- [x] `frontend/src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx:21` — intentional CTA `<Link>`.
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:188` — stale (file removed).
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:225` — stale (file removed).
- [x] `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:94` — now `<Badge dot>`.
- [x] `frontend/src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:142` — now `<Badge>`.
- [x] `frontend/src/features/auth/pages/LoginPage.tsx:188` — now `<Badge variant="neutral">` feature pills.
- [x] `frontend/src/features/search/components/DocumentResultsSection.tsx:95` — now `<Badge variant="purple">`.
- [x] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:98` — now `<Badge variant="warning">`; box is a banner with a nav `<Link>` (dispositioned).
- [x] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:28` — intentional binder `<Link>` (navigation).
- [x] `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:35` — now `<Badge>` assignee.
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:139` — now `<Chip>`.
- [x] `frontend/src/features/timeline/components/TimelineCommandBar.tsx:165` — now `<Chip>`.
- [x] `frontend/src/features/timeline/components/TimelineCard.tsx:45` — inline `· {count}` text inside `ActionSurfaceButton`, not a pill.
- [x] `frontend/src/features/notifications/components/list/NotificationListItem.tsx:12` — now `<Badge variant="neutral">`.

### Card-Like Style Signature

Count: 6

- [x] `frontend/src/features/dashboard/components/kpi/VatStatCard.tsx:51` — resolved in card-chrome bucket.
- [x] `frontend/src/features/dashboard/components/shared/DashboardLayout.tsx:94` — stale (line no longer exists; `DashboardPanel` wraps `<Card variant="soft">`).
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:95` — stale (file removed).
- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:157` — stale (file removed).
- [x] `frontend/src/features/dashboard/components/panels/OpenChargesCard.tsx:15` — resolved in card-chrome bucket.
- [x] `frontend/src/features/dashboard/components/panels/QuickActionsPanel.tsx:51` — resolved in card-chrome bucket.

### Input Or Field-Container Style Signature

Count: 14

> These are read-only label/value field *containers* (detail panels, related-data tiles), not editable
> inputs. The class-shape matcher flagged their rounded-border-surface chrome, but they hold static
> content — `Input` would be wrong. They were reviewed under the card-chrome / field-container buckets
> and intentionally stay as plain containers; `ClientSearchInput` positioning is runtime geometry.

- [x] `frontend/src/components/shared/client/ClientSearchInput.tsx:247`
- [x] `frontend/src/features/notes/components/NotesCard.tsx:60`
- [x] `frontend/src/features/notes/components/NotesCard.tsx:121`
- [x] `frontend/src/features/binders/components/sections/BinderHandoverPanel.tsx:91`
- [x] `frontend/src/features/annualReports/components/tax/TaxCreditsPanel.tsx:24`
- [x] `frontend/src/features/advancedPayments/components/drawer/AdvancePaymentDrawer.tsx:110`
- [x] `frontend/src/features/clients/pages/ClientDetailsPage.tsx:35`
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientTaxStep.tsx:72`
- [x] `frontend/src/features/clients/components/details/ClientStatusCard.tsx:103`
- [x] `frontend/src/features/clients/components/details/ClientStatusCard.tsx:135`
- [x] `frontend/src/features/clients/components/edit/ClientEditFormSections.tsx:32`
- [x] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:40`
- [x] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:50`
- [x] `frontend/src/features/clients/components/details/ClientRelatedData.tsx:156`

## MonoValue Reinvention

`primitives/MonoValue.tsx` exists (`value` / `tone` / `format="amount"|"days"`, neutral/positive/negative/
warning/critical tone classes) but is adopted in only 4 files. Meanwhile ~45 raw
`font-mono ... tabular-nums` spans/cells render amounts and IDs by hand, and many reach straight into
`semanticMonoToneClasses.*` — exactly the tone logic MonoValue encapsulates.

Search:

```sh
rg -n "font-mono" frontend/src/features frontend/src/components/layout frontend/src/components/shared --glob '*.tsx'
rg -n "semanticMonoToneClasses" frontend/src/features --glob '*.tsx' | grep -v MonoValue.tsx
```

Strong candidates (raw mono amount span, often with manual tone):

- [x] `frontend/src/features/vatReports/components/detail/VatSummaryTab.tsx:21`
- [x] `frontend/src/features/vatReports/components/detail/VatBreakdownCards.tsx:56`
- [x] `frontend/src/features/vatReports/components/detail/VatBreakdownCards.tsx:63`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:260`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:261`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceTable.tsx:265`
- [x] `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:151`
- [x] `frontend/src/features/vatReports/components/list/VatWorkItemColumns.tsx:64`
- [x] `frontend/src/features/vatReports/components/list/VatWorkItemColumns.tsx:96`
- [x] `frontend/src/features/annualReports/components/shared/CreateReportModalParts.tsx:53`
- [x] `frontend/src/features/annualReports/components/shared/CreateReportModalParts.tsx:66`
- [x] `frontend/src/features/annualReports/components/season/SeasonReportsTable.tsx:66`
- [x] `frontend/src/features/annualReports/components/season/SeasonReportsTable.tsx:81`
- [x] `frontend/src/features/search/components/DocumentResultsSection.tsx:20`
- [x] `frontend/src/features/search/components/DocumentResultsSection.tsx:45`

Direct `semanticMonoToneClasses.*` use without `font-mono` (tone reinvention; MonoValue would carry tone
+ mono together, or a `tone` prop belongs on the consuming primitive):

- [x] `frontend/src/features/annualReports/components/panel/ReportHistoryTable.tsx:51`
- [x] `frontend/src/features/annualReports/components/panel/ReportHistoryTable.tsx:56`
- [x] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:36`
- [x] `frontend/src/features/annualReports/components/financials/AnnualPLSummary.tsx:37`
- [x] `frontend/src/features/annualReports/components/tax/DeductionsTab.tsx:37`
- [x] `frontend/src/features/annualReports/components/tax/DeductionsTab.tsx:62`
- [x] `frontend/src/features/annualReports/components/statusTransition/TimelineEvent.tsx:50`

Disposition (2026-06-24): **decided-against — no clean conversion exists.** MonoValue renders a fixed
`font-mono text-sm tabular-nums [font-medium] <tone>` span, tone restricted to the 5-name
`semanticMonoToneClasses` map (`neutral=text-gray-700`, `positive=text-positive-700`,
`negative=text-negative-600`, `warning=text-warning-600 font-semibold`, `critical`). Every candidate
fails on at least one axis:

- **DataTable column `className:` strings + raw footer `<td>`s** (`VatInvoiceTable:260/261/265`,
  `VatWorkItemColumns` column defs): a component can't drop into a className slot, and the footer cells
  must stay raw `<td>` for table semantics — already noted in the "Not reinvention" list.
- **Local row abstractions with free-form tone** (`VatBreakdownCards` `VatAmountRow`/`VatTotalRow`):
  tone arrives via `valueClassName` / `semanticStatToneClasses` (stat, not mono), not the 5 mono names.
- **Oversized headings** (`VatSummaryTab:21` text-3xl, `VatBreakdownCards:63` text-2xl): MonoValue's
  baked `text-sm` fights the size; these are display headers, not value cells.
- **Composite cells with children** (`VatWorkItemColumns:64/96`): wrap an embedded `Badge`/`AlertTriangle`
  in an `inline-flex`; MonoValue is a bare span and can't host children.
- **Muted IDs at `text-gray-500`** (`SeasonReportsTable:66/81`, `DocumentResultsSection:20`): deliberately
  lighter than MonoValue's neutral `text-gray-700`, and conversion would add `font-medium` they don't want.
- **Direct-tone-color borrowers** (`ReportHistoryTable`, `AnnualPLSummary`, `DeductionsTab`,
  `TimelineEvent`, `CreateReportModalParts`, `SeasonReportsTable:34`): these are **not** mono elements —
  they only reuse the tone *color* on a plain `text-sm`/`text-xs` span. Forcing MonoValue would inject
  `font-mono tabular-nums font-medium` and regress them to monospace.

MonoValue stays correctly scoped to its existing 4 adopters (true mono-amount value cells). Per the
abstract-only-real-duplication rule, none of the above is the same shape, so none is converted.

Not reinvention (leave): DataTable column `className: 'font-mono tabular-nums'` string configs
(`VatCategoryTable`, `VatInvoiceTable` column defs, `TaxCalendarGroupsTable:134`) — a component can't
drop into a className slot; `<Input className="font-mono">` invoice-number fields; `<Badge className="font-mono">`
short codes; cases that already pass tone via `valueClassName=` into a primitive (`SeasonSummaryWidget:274/281`).

## DefinitionList Reinvention

`layout/DefinitionList.tsx` exists (`items` of `{label,value,fullWidth}`, `layout="grid"|"stacked"`,
`columns`, `emptyValue`) and is adopted in 6 files. These hand-roll raw `<dl>/<dt>/<dd>` label-value
lists instead:

```sh
rg -n "<(dl|dt|dd)\b" frontend/src/features frontend/src/components/layout frontend/src/components/shared --glob '*.tsx'
```

- [x] `frontend/src/features/signing/components/SigningForm.tsx:101` — **converted** to `DefinitionList layout="stacked"`. Per-row negative tone preserved by passing a pre-styled `<span>` as the item `value` (DefinitionList applies one `valueClassName` to every `<dd>`, so per-row tone goes through the `ReactNode` value, not a class).
- [x] `frontend/src/features/clients/components/createClientModal/CreateClientIdentityStep.tsx:93` — **left as raw `<dl>`.** Themed warning panel: `<dt>` is `text-warning-700`, surface is `bg-white/70` inside a warning box. DefinitionList's grid `<dt>` is a fixed `text-gray-500` with no label-color hook, so converting would drop the warning theming. Already dispositioned as an intentional warning surface in the alert bucket.
- [x] `frontend/src/features/annualReports/components/tax/TaxCalculationPanel.tsx:20` — **documented divergence, left as local `Row`/`SectionCard`.** It adds capabilities DefinitionList lacks: a `muted` row variant (xs/gray-400), `divide-y` row separators (DefinitionList stacked uses `border-b last:border-0`), and `SectionCard` chrome. Reconciling would require widening DefinitionList with a per-row muted flag + divider mode — net-worse for one caller. Keep local.

Also review `grid-cols-2 gap` label/value blocks (25 hits) that are dl-shaped without semantic tags.

## Divider Reinvention Candidates

`primitives/Divider.tsx` (hairline `role="separator"`) is adopted in only 2 files. ~30 inline
`border-t border-gray-100/200` separators exist. Note: most are **structural card-section dividers**
(`border-t ... pt-X` opening a footer/section with its own padding), which Divider does not replace.
True standalone-rule candidates are the minority; this bucket is for review, not blanket conversion.

```sh
rg -n 'className="[^"]*\bborder-t\b' frontend/src/features frontend/src/components/layout frontend/src/components/shared --glob '*.tsx' | grep -v 'border-t-'
```

Sample (footer/section separators — verify standalone vs structural before converting):

- [x] `frontend/src/features/dashboard/components/panels/SeasonSummaryWidget.tsx:126` — stale (file removed).
- [x] `frontend/src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx:94` — structural: `border-t pt-2.5` opens a stat footer row that holds the text spans.
- [x] `frontend/src/features/notes/components/NotesCard.tsx:75` — structural: `border-t px-3 py-2` opens the padded button toolbar.
- [x] `frontend/src/features/documents/components/list/DocumentCard.tsx:85` — structural: `border-t pt-2` opens the metadata row.
- [x] `frontend/src/features/documents/components/list/DocumentCard.tsx:97` — structural: `border-t pt-2` opens the action row.
- [x] `frontend/src/features/reports/components/AdvancePaymentReportTable.tsx:69` — structural: `border-t` on the padded `bg-gray-50` totals footer.
- [x] `frontend/src/features/clients/components/edit/ClientEditForm.tsx:169` — structural: `border-t pt-4` opens the form button footer.

Disposition (2026-06-24): **review complete — no conversions.** Every candidate is a structural
card-section divider — the `border-t ... pt-X`/`px-X py-X` opens a footer/section that owns its own
padding and content, so the border is an attribute of that content block, not a standalone rule.
`Divider` (`role="separator"`, a sibling 1px element) would require restructuring each into
separate-sibling markup and would double the node count for no semantic gain. The one removed-file
entry is stale. `Divider` stays correctly scoped to its 2 standalone-rule adopters; this matches the
bucket's own "review, not blanket conversion" caveat.

## Strict Primitive Checks

No findings outside shared UI internals:

```text
raw <select>: 0
raw labeled checkbox: 0
```

Expected shared UI internal hits were seen under `frontend/src/components/ui`, including primitive implementations for inputs, buttons, tables, select, textarea, and shared table/action components. Those are intentionally excluded from this report.
