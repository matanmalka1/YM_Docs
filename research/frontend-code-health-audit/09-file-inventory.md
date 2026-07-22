# Frontend file inventory

Audit date: 2026-07-22. Scope: every file under `frontend/src/**`, including tests, styles, README material, and the one ignored Finder artifact. This inventory contains **exactly 870 unique file rows**: 869 git-visible files and one ignored `.DS_Store`.

Classification is conservative and file-level. A live file can be marked `keep but refactor` because it contains removable members; that does not make the whole file dead. Likewise, `wrapper` and `duplicate implementation` identify live code whose structure should be simplified, not current deletion candidates. Production reachability was traced from `src/main.tsx` through TypeScript imports, re-exports, CSS imports, and dynamic imports; tests, routes, feature barrels, and Knip output were inspected before classification.

An unrelated untracked `.claude/worktrees/table-refactor/` directory exists in the frontend repository but is outside `frontend/src/**` and this inventory. A full Knip invocation lists that copied tree as unused; filtering its JSON result to canonical `src/` paths yields no unused file.

## Totals

| Classification | Files |
| --- | ---: |
| keep | 758 |
| keep but refactor | 89 |
| candidate for deletion | 1 |
| duplicate implementation | 2 |
| wrapper | 3 |
| misplaced | 17 |
| requires runtime verification | 0 |
| **Total** | **870** |

The only whole-file deletion candidate is `src/features/notifications/.DS_Store`. All 840 production TypeScript/TSX/CSS files are runtime-reachable, so none is classified as a dead production file. `Misplaced` means the live file has a clear different owner; it does not imply deletion. `useBusinessesForClient.ts` is included there even though the move should also remove its dead callback/effect.

## Inventory

| File | Classification | Evidence / rationale |
| --- | --- | --- |
| `src/api/auth-session.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/api/auth.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/api/client.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/api/core-endpoints.ts` | keep but refactor | Contains live auth endpoints but dead CORE_ENDPOINTS.health/info members; remove only those unused properties. |
| `src/api/queryParams.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/errors/AppErrorBoundary.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/ClientSidebar/ClientSidebar.grouping.ts` | misplaced | Client-domain grouping belongs with the connected client-navigation widget, not generic app layout (`SQ-07`). |
| `src/components/layout/ClientSidebar/ClientSidebar.labels.ts` | misplaced | Imports client/user domain labels for the connected client-navigation widget (`SQ-07`). |
| `src/components/layout/ClientSidebar/ClientSidebar.tsx` | misplaced | Mixes client search/grouping/cards with account/logout and responsive layout; move the connected client widget to clients ownership (`SQ-07`). |
| `src/components/layout/ClientSidebar/ClientSidebarClientCard.tsx` | misplaced | Client-domain card and route presentation belongs with the clients-owned sidebar widget (`SQ-07`). |
| `src/components/layout/ClientSidebar/useClientSidebarClients.ts` | misplaced | Connected clients API query is hidden under generic layout and deep-imports the clients API (`SQ-07`). |
| `src/components/layout/ClientSidebar/useClientSidebarFocusTrap.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/Navbar/Navbar.constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/Navbar/Navbar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/Navbar/Navbar.utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/Navbar/NavbarMoreMenu.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/Navbar/NavbarPrimaryNav.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/NotificationBell.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/PageContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/PageHeader.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/components/layout/PageHeader.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/layout/PageLayout.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/shared/client/ClientPickerField.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/shared/client/ClientSearchInput.tsx` | misplaced | A shared widget directly owns clients API querying/debounce/error state; move the connected field to clients and retain only a pure combobox primitive (`SQ-02`). |
| `src/components/shared/client/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/shared/client/useClientPickerState.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/feedback/InlineState.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/feedback/StateCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/feedback/Timeline.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/feedback/UnsavedChangesGuard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/feedback/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/filters/ClientPickerFilter.tsx` | keep but refactor | The generic filter system has a client-domain branch; replace it with a clients-owned composition or typed slot (`SQ-02`). |
| `src/components/ui/filters/FilterPanel.tsx` | keep but refactor | Runtime-reachable 268-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/components/ui/filters/SearchFilter.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/filters/filterBadges.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/filters/types.ts` | keep but refactor | `ClientPickerFieldDef` makes the generic filter DSL know client-domain result shapes (`SQ-02`). |
| `src/components/ui/index.ts` | wrapper | Pure export-star aggregator over UI sub-barrels; live as an import surface but adds no runtime behavior and broadens ownership/indirection. |
| `src/components/ui/inputs/DatePicker.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/DatePickerCalendar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/DatePickerInlineSelect.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/FormField.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/HiddenFileInput.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/Input.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/PasswordInput.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/Select.tsx` | keep but refactor | Advertises native select attributes but renders a custom listbox and drops unsupported props/ref (`SQ-11`). |
| `src/components/ui/inputs/SelectDropdown.helpers.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/components/ui/inputs/SelectDropdown.helpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/SelectDropdown.tsx` | keep but refactor | Runtime-reachable 306-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/components/ui/inputs/Textarea.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/inputs/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/layout/DefinitionList.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/DetailTabPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/MetaStrip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/OverlayContainer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/PageLoading.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/PageStateGuard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/SectionHeader.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/StatsCard.tsx` | keep but refactor | Live shared card has five wholly unused configuration branches plus generic count-up animation; delete dead surface and keep the used card contract focused. |
| `src/components/ui/layout/ToolbarContainer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/layout/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/overlays/Alert.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/AlertBanner.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/ConfirmDialog.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/DetailDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/DrawerPrimitives.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/Modal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/ModalFormActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/OverlayPortalContext.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/overlays/useDismissibleLayer.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/useModalDialog.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/useOverlayDismiss.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/overlays/useUnsavedChangesGuard.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/ActionSurface.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Badge.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Button.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Card.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/CarouselDots.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Checkbox.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Chip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/DismissBackdrop.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Divider.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/GroupSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/InlineLink.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/MonoValue.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/ProgressBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/SegmentedControl.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/SkeletonBlock.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Spinner.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/StatusBadge.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/Tooltip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/primitives/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/table/README.md` | keep | Co-located module documentation, not runtime code. |
| `src/components/ui/table/columns/EmptyCell.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/columns/commonColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/columns/tableSelection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/core/DataTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/core/PaginatedDataTable.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/components/ui/table/core/PaginatedDataTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/core/TableSkeleton.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/core/tableStyles.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/core/tableTypes.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/grouping/GroupedPeriodRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/grouping/MonthlyAccordionGroup.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/grouping/groupedPeriodRow.utils.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/components/ui/table/grouping/groupedPeriodRow.utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/components/ui/table/toolbar/ActiveFilterBadges.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/toolbar/BulkSelectionToolbar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/toolbar/PaginationCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/components/ui/table/toolbar/RowActions.tsx` | keep but refactor | Runtime-reachable 408-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/constants/filterOptions.constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/constants/pagination.constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/constants/periodOptions.constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/constants/searchPlaceholders.constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/api/advancedPayments.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/advancedPayments/api/queryKeys.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/advancedPayments/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/create/CreateAdvancePaymentFlow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/create/CreateAdvancePaymentModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/create/GenerateScheduleModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentContextCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentDetailView.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentEditableSections.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentFullPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentReadonlySections.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/panel/AdvancePaymentSummaryStrip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/stats/AdvancePaymentsStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/table/AdvancePaymentBatchColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/table/AdvancePaymentBatchContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/table/AdvancePaymentBatchRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/components/table/AdvancePaymentBatchesList.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentBatchRows.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentBatches.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentDetailForm.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentDetailPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePayments.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvancePaymentsPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useAdvanceRateInsights.ts` | keep but refactor | Wraps the residual tax-profile facade; its update action/loading outputs have no consumer and should disappear during ownership cleanup (`SQ-26`). |
| `src/features/advancedPayments/hooks/useClientAdvancePaymentsTab.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useCreateAdvancePayment.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/hooks/useGenerateSchedule.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/advancedPayments/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/pages/AdvancePaymentDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/pages/AdvancePaymentsPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/pages/ClientAdvancePaymentsTab/ClientAdvancePaymentsCards.tsx` | misplaced | Presentational child of a client-details tab, not a route page; move with the tab to `components/detail` (`SQ-10`). |
| `src/features/advancedPayments/pages/ClientAdvancePaymentsTab/ClientAdvancePaymentsHeader.tsx` | misplaced | Presentational child of a client-details tab, not a route page; move with the tab to `components/detail` (`SQ-10`). |
| `src/features/advancedPayments/pages/ClientAdvancePaymentsTab/ClientAdvancePaymentsStatsSection.tsx` | misplaced | Presentational child of a client-details tab, not a route page; move with the tab to `components/detail` (`SQ-10`). |
| `src/features/advancedPayments/pages/ClientAdvancePaymentsTab/ClientAdvancePaymentsTab.tsx` | misplaced | Reusable client-details widget is stored under route-only `pages/`; no router consumes it (`SQ-10`). |
| `src/features/advancedPayments/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/utils/advancePaymentComponentUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/advancedPayments/utils/advancePaymentUtils.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/advancedPayments/utils/advancePaymentUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/api/annualReports.api.ts` | keep but refactor | Live API object contains dead listReportCharges transport method; remove its cascade-only contract/endpoint. |
| `src/features/annualReports/api/annualReports.financials.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/api/annualReports.season.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/api/annualReports.status.api.ts` | keep but refactor | Live API object contains unconsumed updateDeadline mutation. |
| `src/features/annualReports/api/annualReports.tax.api.ts` | keep but refactor | Live API object contains unconsumed getAdvancesSummary query. |
| `src/features/annualReports/api/contracts.ts` | keep but refactor | Live contracts file includes AnnualReportChargesResponse used only by dead listReportCharges. |
| `src/features/annualReports/api/endpoints.ts` | keep but refactor | Live endpoint map contains dead transition plus endpoints referenced only by dead API methods. |
| `src/features/annualReports/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/annualReports/api/queryKeys.ts` | keep but refactor | Live key factory contains dead reportCharges, reportDetails, and advancesSummary members. |
| `src/features/annualReports/api/utils.ts` | misplaced | API-layer module owns badge types, Hebrew labels, schedule labels, and Tailwind progress config; move display policy to feature constants/presentation (`SQ-28`). |
| `src/features/annualReports/components/annex/AnnexCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexDataPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexDataTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexEmptyState.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexFieldInput.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexHeader.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnexRowForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/AnnualReportAnnexesTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/annex/ScheduleAddForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AddExpenseLineForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AnnualPLSummary.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AnnualReportAddIncomeLineForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AnnualReportFinancialSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AnnualReportVatAutoPopulateControls.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/AnnualReportVatAutoPopulateResultPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/EditExpenseLineForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/EditIncomeLineForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/FinancialLineFormParts.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/FinancialLineRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/IncomeExpenseTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/financials/MultiYearPLChart.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/AnnualReportFullPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/AnnualReportOverviewTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/AnnualReportSectionContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/AnnualReportSummaryStrip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/AnnualReportTimelineTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/ReadinessCheckPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/ReportAlertBanners.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/ReportHistoryTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/panel/ReportMetaGrid.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/season/SeasonProgressBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/season/SeasonReportsTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/shared/ClientAnnualReportsTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/shared/ClientYearComparisonModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/shared/CreateReportModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/shared/CreateReportModalParts.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/shared/OverdueBanner.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/statusTransition/AmendReportModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/statusTransition/StatusTransitionPanelRoot.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/statusTransition/TransitionDetailsForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/statusTransition/TransitionTargetSelector.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/AnnualReportDetailForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/DeductionsTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/TaxBracketsTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/TaxCalculationTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/TaxCalculatorInputs.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/components/tax/TaxCreditsPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/annexConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/annexSchema.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/annexTextConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/financialConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/panelConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/reportConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/sharedConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/statusTransitionConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/submissionMethodOptions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/constants/taxConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useAnnexDataPanel.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useAnnualPLSummary.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useAnnualReportDetailPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useAnnualReportsPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useClientAnnualReportsTab.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useCreateReport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useFinancialLineForm.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useIncomeExpenseMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useIncomeExpenseTab.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useReportDetail.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useReportMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useReportSchedules.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useSeasonDashboard.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useStatusTransitionPanel.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/hooks/useTaxCalculationTab.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/annualReports/messages.ts` | keep but refactor | Runtime-reachable 276-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/annualReports/pages/AnnualReportDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/pages/AnnualReportsPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/annualReports/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/types.ts` | keep but refactor | Live shared types, but CURRENT_YEAR at line 7 has no consumer and should be deleted. |
| `src/features/annualReports/utils/annexHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/annualReportsUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/financialHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/panelHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/reportSummaryHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/sharedHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/statusTransitionHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/annualReports/utils/taxHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/api/audit.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/audit/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/components/AuditTrailTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/components/EntityAuditTrailSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/hooks/useEntityAuditTrail.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/hooks/useEntityAuditTrailSection.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/audit/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/audit/utils/auditFormatters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/auth/components/AuthPageShell.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/auth/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/auth/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/auth/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/auth/pages/ForgotPasswordPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/auth/pages/LoginPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/auth/pages/ResetPasswordPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/auth/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/api/authorityContacts.api.ts` | keep but refactor | Live API object contains dead getAuthorityContact; edit flow uses list-row data. |
| `src/features/authorityContacts/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/authorityContacts/api/queryKeys.test.ts` | keep but refactor | Test asserts a query-key member with no production consumer; delete/update with that dead member. |
| `src/features/authorityContacts/api/queryKeys.ts` | keep but refactor | detail key has no runtime consumer and is retained only by its unit test. |
| `src/features/authorityContacts/api/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/form/AuthorityContactFormFields.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/form/AuthorityContactModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/list/AuthorityContactRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/list/AuthorityContactsListCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/list/AuthorityContactsListContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/components/shared/AuthorityContactsCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/helpers.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/authorityContacts/helpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/hooks/useAuthorityContactForm.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/hooks/useAuthorityContacts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/hooks/useAuthorityContactsCardState.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/authorityContacts/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/authorityContacts/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/api/binders.api.ts` | keep but refactor | Live API object contains dead getOpenBinders and updateIntake methods. |
| `src/features/binders/api/contracts.ts` | keep but refactor | Live binder contracts include BinderIntakeUpdatePayload used only by dead updateIntake. |
| `src/features/binders/api/endpoints.ts` | keep but refactor | Live endpoint map contains bindersOpen/binderIntakeById used only by dead API methods. |
| `src/features/binders/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/binders/api/queryKeys.ts` | keep but refactor | Live key factory contains unused open and forClient members. |
| `src/features/binders/components/dialogs/BinderHandoverPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/dialogs/BindersPageDialogs.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderActionButtons.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderAuditSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderDetailDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderDetailsPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderDocumentsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderIntakesSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderPeriodFields.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/drawer/BinderReceivePanel.tsx` | keep but refactor | Runtime-reachable 274-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/binders/components/list/BinderRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/list/BindersColumns.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/binders/components/list/BindersColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/list/BindersStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/components/shared/ClientBindersTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useBinderDetail.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useBinderMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useBinderSelection.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useBindersFilters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useBindersPage.ts` | keep but refactor | Runtime-reachable 300-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/binders/hooks/useBindersPageDialogs.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/hooks/useReceiveBinderDrawer.ts` | keep but refactor | Runtime-reachable 317-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/binders/index.ts` | keep but refactor | Intentional feature boundary, but six constants/types are over-exported with no external consumer. |
| `src/features/binders/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/pages/BindersPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/binders/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/binders/types.ts` | keep but refactor | Live binder contracts include ListOpenBindersParams used only by dead getOpenBinders. |
| `src/features/binders/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/api/endpoints.ts` | keep but refactor | Live endpoint map contains businessRestore used only by dead clientsApi.restoreBusiness. |
| `src/features/businesses/components/BusinessDetailsCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/hooks/useBusinessDetails.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/businesses/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/pages/BusinessDetailsPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/businesses/utils/businessUtils.tsx` | misplaced | Utility module constructs rendered Link JSX; presentation ownership is hidden outside components/pages. |
| `src/features/charges/api/charges.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/api/index.ts` | keep but refactor | Intentional API barrel over-exports CHARGE_ROUTES; callers import its owner directly. |
| `src/features/charges/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/detail/ChargeActionButtons.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/detail/ChargeDetailPanel.tsx` | keep but refactor | Runtime-reachable 259-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/charges/components/form/ChargeEditModal.tsx` | duplicate implementation | Live edit modal; duplicates the charge core-field form markup and option wiring found in ChargesCreateModal. Consolidate only the shared field section, not mutation-specific behavior. |
| `src/features/charges/components/form/ChargesCreateModal.tsx` | duplicate implementation | Live create modal; duplicates the charge core-field form markup and option wiring found in ChargeEditModal. Consolidate only the shared field section, not mutation-specific behavior. |
| `src/features/charges/components/list/ChargeBulkToolbar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/list/ChargeClientCell.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/list/ChargeColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/list/ChargeRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/list/ChargesStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/list/ChargesTableBlock.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/components/shared/ClientChargesTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/hooks/useChargeActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/hooks/useChargeCreateMutation.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/hooks/useChargeDetailsPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/hooks/useChargesFilters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/hooks/useChargesPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/index.ts` | keep but refactor | Intentional feature boundary over-exports ChargeListItem with no external consumer. |
| `src/features/charges/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/pages/ChargeDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/pages/ChargesPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/charges/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/utils/chargeCache.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/charges/utils/chargeCache.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/utils/chargeHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/charges/utils/chargeUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/api/clients.api.ts` | keep but refactor | Live API object contains unconsumed restoreBusiness. |
| `src/features/clients/api/contracts.ts` | keep but refactor | Runtime-reachable 271-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/clients/api/endpoints.ts` | keep but refactor | Live route map contains six unconsumed client-detail route builders. |
| `src/features/clients/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/clients/api/queryKeys.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/clients/api/queryKeys.ts` | keep but refactor | Live key factory contains dead taxProfile and firstBusiness members. |
| `src/features/clients/components/detail/ClientBusinessesCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientDetailsOverviewTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientDetailsTabBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientDetailsTabContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientInfoSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientInfoSectionParts.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientRelatedData.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/ClientStatusCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/detail/DeleteClientModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/dialogs/DeletedClientDialog.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/drawer/ClientEditDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/ClientEditForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/ClientEditFormSections.tsx` | keep but refactor | Runtime-reachable 282-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/clients/components/form/CreateBusinessModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientBusinessStep.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientIdentityStep.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientModalFooter.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientReviewStep.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientStepContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientStepIndicator.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/CreateClientTaxStep.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/form/createClientImpactIcons.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/list/ClientColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/list/ClientRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/components/list/ClientsStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useBusinessActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientAuthorityContacts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientCreationImpact.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientDetailsActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientQuery.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientsFilters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useClientsPage.ts` | keep but refactor | Runtime-reachable 266-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/clients/hooks/useDeletedClientConflict.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useFilterClient.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useFirstBusinessId.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/hooks/useIdNumberConflict.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/index.ts` | keep but refactor | Intentional feature boundary over-exports six constants/parsers/types with no external consumer. |
| `src/features/clients/messages.ts` | keep but refactor | Runtime-reachable 298-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/clients/pages/ClientAdvancePaymentDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/pages/ClientAnnualReportDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/pages/ClientChargeDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/pages/ClientDetailsPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/clients/pages/ClientVatWorkItemDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/pages/ClientsPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/clients/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/utils/buildClientUpdatePayload.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/utils/clientEditImpact.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/utils/clientErrors.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/utils/createClientFormUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/clients/utils/createClientSteps.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/api/correspondence.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/correspondence/api/queryKeys.ts` | keep but refactor | forBusiness has no consumer; the implemented flow is client-wide. |
| `src/features/correspondence/components/CorrespondenceCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/components/CorrespondenceEntry.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/components/CorrespondenceModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/hooks/useCorrespondence.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/correspondence/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/correspondence/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/api/dashboard.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/dashboard/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/kpi/DashboardStatsSkeleton.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/kpi/VatStatCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/panels/AttentionBoard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/panels/OpenChargesCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/panels/QuickActionsPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/panels/RecentActivityPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/panels/SeasonInsightsCarousel.tsx` | keep but refactor | Runtime-reachable 276-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/dashboard/components/panels/UpcomingDeadlinesPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/shared/DashboardLayout.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/components/shared/DashboardOnboardingEmptyState.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/hooks/useDashboardCreateModals.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/hooks/useDashboardPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/hooks/useSeasonSummary.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/dashboard/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/dashboard/pages/DashboardPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/dashboard/utils/dashboardUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/api/documents.api.ts` | keep but refactor | Live API object contains unconsumed listByAnnualReport. |
| `src/features/documents/api/endpoints.ts` | keep but refactor | documentsByAnnualReport is referenced only by dead listByAnnualReport. |
| `src/features/documents/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/documents/api/queryKeys.ts` | keep but refactor | byAnnualReport has no consumer and pairs with the dead transport method. |
| `src/features/documents/components/detail/DocumentPreviewModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/detail/DocumentVersionsPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/form/DocumentEditCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/form/DocumentsUploadCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/list/DocumentCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/list/DocumentsDataCards.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/list/DocumentsFilterPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/components/shared/ClientDocumentsTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/hooks/useClientDocumentsTab.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/hooks/useDocumentCardActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/hooks/useDocumentPreviewDownload.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/hooks/useDocumentUpload.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/documents/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/utils/documentsDataCardsUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/documents/utils/documentsUploadCardHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/importExport/api/importExport.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/importExport/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/importExport/components/ImportExportModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/importExport/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/importExport/hooks/useImportExport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/importExport/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/importExport/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/invoices/api/invoices.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/components/ChargeInvoiceSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/hooks/useChargeInvoice.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/invoices/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/invoices/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/notes/api/notes.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/components/NotesCard.tsx` | keep but refactor | Runtime-reachable 278-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/notes/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/hooks/useEntityNotes.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/hooks/useNotesResource.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notes/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/notes/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/.DS_Store` | candidate for deletion | Ignored macOS Finder metadata (6,148 bytes), not source code, not tracked, and has no runtime or tooling consumer. |
| `src/features/notifications/api/contracts.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/notifications/api/contracts.ts` | keep but refactor | Mixes transport shapes with manual capability lists and duplicated UI/backend trigger labels (`CT-29`). |
| `src/features/notifications/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/notifications/api/notifications.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/api/queryKeys.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/notifications/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/components/drawer/NotificationDetailDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/components/drawer/NotificationDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/components/form/SendNotificationModal.tsx` | keep but refactor | Runtime-reachable 253-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/notifications/components/list/NotificationListItem.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/components/list/NotificationsColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/components/shared/NotificationsTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/hooks/useNotificationBell.ts` | wrapper | Live one-consumer wrapper that only renames a local boolean useState into open/close callbacks; it adds no notification contract, permission, analytics, persistence, or accessibility behavior. |
| `src/features/notifications/hooks/useNotificationDetail.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/hooks/useNotifications.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/hooks/useNotificationsPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/hooks/useSendNotification.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/notifications/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/pages/NotificationsPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/notifications/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/reports/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/api/reports.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/components/AdvancePaymentReportTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/components/AgingReportCards.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/components/AgingReportHeader.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/components/AnnualReportStatusTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/hooks/useAdvancePaymentReport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/hooks/useAgingReport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/hooks/useAnnualReportStatusReport.ts` | keep but refactor | Live report query hook stores an immutable derived current year and returns an unconsumed setter; remove the inert controlled/uncontrolled state API. |
| `src/features/reports/hooks/useVatComplianceReport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/reports/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/pages/AdvancePaymentReportView.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/pages/AgingReportView.tsx` | keep but refactor | Live standalone report page retains an abandoned embedded prop and unreachable embedded-only rendering branches; remove only the dead mode. |
| `src/features/reports/pages/AnnualReportStatusView.tsx` | keep but refactor | Live report page exposes an unconsumed controlled taxYear prop; simplify after confirming current-year-only product intent. |
| `src/features/reports/pages/VatComplianceReportView.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/reports/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/api/contracts.ts` | keep but refactor | Two internal shape types are exported without any external type consumer. |
| `src/features/search/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/search/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/api/search.api.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/api/search.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/components/SearchClientMatches.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/components/SearchMatchRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/components/SearchMatchesSection.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/components/SearchMatchesSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/components/SearchResultsSection.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/components/SearchResultsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/components/SearchToolbar.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/components/SearchToolbar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/hooks/useSearchPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/search/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/pages/SearchPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/utils/searchMatches.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/utils/searchMatches.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/search/utils/searchUrlValues.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/search/utils/searchUrlValues.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/signatureRequests/api/queryKeys.ts` | keep but refactor | businessNamesBatch has no consumer; current dashboard naming uses loaded data. |
| `src/features/signatureRequests/api/signatureRequests.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/drawer/SignatureRequestAuditDrawer.test.tsx` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/signatureRequests/components/drawer/SignatureRequestAuditDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/form/CreateSignatureRequestModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/list/SignatureRequestRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/list/SignatureRequestRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/shared/SignatureRequestsCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx` | keep but refactor | Live component imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/signatureRequests/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/hooks/useClientSignatureRequests.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/hooks/usePendingSignatureRequests.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/hooks/useSignatureRequestActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/signatureRequests/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signatureRequests/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/components/SigningForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/components/SigningStatus.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/hooks/useSigningPageState.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/signing/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/signing/pages/SigningPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/signing/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/tasks/api/queryKeys.ts` | keep but refactor | lists root has no consumer; list/detail/all are the live key surface. |
| `src/features/tasks/api/tasks.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/form/TaskDetailsFields.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/form/TaskModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/form/TaskSourceSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/list/TaskListColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/list/TasksListPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/list/TasksListSummary.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/components/shared/ClientTasksTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/constants/labels.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/constants/pageConstants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useBulkAssignTasks.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useBulkCompleteTasks.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useTaskActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useTaskFilters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useTaskSourcePicker.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useTasks.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/hooks/useTasksPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/tasks/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/pages/TasksPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/schemas.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/tasks/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/utils/taskDisplay.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/utils/taskFilters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/tasks/utils/taskFormatters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/api/index.ts` | misplaced | Uses export-star and documents a deferred split of the mixed taxCalendar API module. |
| `src/features/taxCalendar/api/taxCalendar.api.ts` | misplaced | Combines contracts, labels, query keys, endpoints, and transport in one API file; explicit project TODO confirms the ownership split. |
| `src/features/taxCalendar/components/list/TaxCalendarFiltersBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/components/list/TaxCalendarGroupsContent.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/components/list/TaxCalendarGroupsTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/components/list/TaxCalendarStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/components/shared/ClientTaxCalendarList.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/components/shared/ClientTaxCalendarTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/helpers.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/taxCalendar/helpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/hooks/useOpenClientTaxCalendarItem.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/hooks/useTaxCalendarGroupItems.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/hooks/useTaxCalendarGroups.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/hooks/useTaxCalendarGroupsPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxCalendar/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/pages/TaxCalendarGroupsPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendar/utils.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/taxCalendar/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxCalendarSettings/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/api/taxCalendarSettings.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/components/TaxCalendarSettingsStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/hooks/useTaxCalendarSettings.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxCalendarSettings/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx` | keep but refactor | Runtime-reachable 362-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/taxDashboard/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxDashboard/api/queryKeys.ts` | keep but refactor | urgentDeadlines has no consumer; only submissions is queried. |
| `src/features/taxDashboard/api/taxDashboard.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/components/TaxSubmissionStats.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/helpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/hooks/useTaxDashboard.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxDashboard/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxDashboard/pages/TaxDashboardPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/taxProfile/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/taxProfile/hooks/useTaxProfile.ts` | keep but refactor | Live tax-profile hook is a client-detail/update adapter with an unsafe response cast and direct dependency on the clients feature root; clarify ownership and type the query result without assertion. |
| `src/features/taxProfile/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/taxProfile/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/timeline/api/queryKeys.ts` | keep but refactor | clientRoot has no consumer; clientEvents and all are the live key surface. |
| `src/features/timeline/api/timeline.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/components/ClientTimelineTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/components/TimelineCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/components/TimelineEventItem.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/components/TimelineFilterPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/components/TimelineMetadata.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/hooks/useClientTimelinePage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/timeline/labels.ts` | keep but refactor | Uses five suppressed private imports across four feature owners and duplicates annual status wording (`SQ-06`, `DU-04`). |
| `src/features/timeline/lib/timelineGroups.ts` | keep but refactor | Contains an unreachable default-open branch for the removed timeline `future` key (`SQ-34`). |
| `src/features/timeline/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/normalize.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/timeline/normalize.ts` | keep but refactor | Returns an always-empty, unconsumed deadline collection and retains a filter key no event producer emits (`SQ-34`). |
| `src/features/timeline/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/timeline/utils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/users/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/api/users.api.ts` | keep but refactor | Live API object contains unconsumed getById; users edit from list data. |
| `src/features/users/components/drawer/AuditLogsDrawer.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/form/CreateUserModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/form/EditUserModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/form/ResetPasswordModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/form/UserFormFields.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/list/UserRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/components/list/UsersColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/hooks/useActiveUserFilterOptions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/hooks/useActiveUserOptions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/hooks/useAdvisorOptions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/hooks/useUsersPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/users/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/users/pages/UsersPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/users/schemas.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/api/contracts.ts` | keep but refactor | Live contracts include three parameter/payload types used only by dead VAT methods. |
| `src/features/vatReports/api/endpoints.ts` | keep but refactor | vatWorkItemsByClient is referenced only by dead listByClient. |
| `src/features/vatReports/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/vatReports/api/mutationKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/api/vatReports.api.ts` | keep but refactor | Live API object contains dead flat-list, generic update, and client-list methods. |
| `src/features/vatReports/components/detail/VatBreakdownCards.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatCategoryTable.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatClientSummaryPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatClientSummaryStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatHistoryTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatInvoiceTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatPeriodCard.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatSummaryTab.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatWorkItemFullPanel.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/detail/VatWorkItemMetaStrip.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/form/VatFileModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/form/VatInvoiceAddForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/form/VatPeriodSelect.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/form/VatSendBackForm.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/form/VatWorkItemsCreateModal.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatInvoiceEditRow.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatInvoiceTable.tsx` | keep but refactor | Runtime-reachable 275-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/vatReports/components/list/VatWorkItemColumns.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatWorkItemRowActions.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatWorkItemSummaryBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatWorkItemsGroupedCards.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/list/VatWorkItemsStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/shared/VatActionButtons.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/shared/VatExportButtons.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/shared/VatFiledBanner.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/components/shared/VatProgressBar.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/constants/vatConstants.ts` | keep but refactor | Live constants file contains unconsumed CATEGORY_TABLE_LABELS. |
| `src/features/vatReports/constants/visualizationTokens.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useActiveVatBinder.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useFileVatReturn.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatClientSummary.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatExport.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatGroupItems.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatInvalidation.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatInvoiceMutations.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatLifecyclePending.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatPeriodOptions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatWorkItemActions.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatWorkItemDetailPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatWorkItemGroups.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatWorkItemPage.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/hooks/useVatWorkItemsPage.ts` | keep but refactor | Runtime-reachable 325-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/vatReports/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/vatReports/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/pages/VatWorkItemDetailPage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/pages/VatWorkItemsPage.tsx` | keep but refactor | Live page imports its own feature root barrel, creating a forbidden self-boundary edge; replace with direct relative imports. |
| `src/features/vatReports/schemas/fileVatReturn.schema.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/schemas/invoice.schema.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/schemas/workItem.schema.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/utils/filters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/utils/vatHelpers.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/features/vatReports/utils/vatHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/vatReports/utils/viewHelpers.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/api/contracts.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/api/endpoints.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/api/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/workQueue/api/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/api/workQueue.api.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/components/WorkQueueStatsSection.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/components/workQueueColumns.tsx` | keep but refactor | Runtime-reachable 265-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/workQueue/constants.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/errorMessages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/hooks/useWorkQueue.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/hooks/useWorkQueueActions.ts` | keep but refactor | Runtime-reachable 264-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/workQueue/hooks/useWorkQueuePage.ts` | keep but refactor | Runtime-reachable 277-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/features/workQueue/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/features/workQueue/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/pages/WorkQueuePage.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/features/workQueue/utils/taskRelationFilter.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/hooks/useBusinessesForClient.ts` | misplaced | Live client-domain query belongs to the clients/business owner; also remove its unconsumed `onAutoSelect` option/effect (`SQ-08`). |
| `src/hooks/useDefaultOpenGroup.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/hooks/useDetailQuery.ts` | wrapper | Live one-consumer generic wrapper around useQuery; it adds only id-based enabling and hides annual-report-specific casts, so inline it into useReportDetail. |
| `src/hooks/useMutationWithToast.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/hooks/useRole.ts` | keep but refactor | Live role hook exposes two capability flags with zero consumers; remove the misleading dead flags and address feature-specific authorization separately. |
| `src/hooks/useSearchDebounce.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/hooks/useSearchParamFilters.ts` | keep but refactor | Live hook; nextFilterParams/resetFilterParams are needed locally but unnecessarily exported. |
| `src/hooks/useTableSelection.ts` | misplaced | One-consumer selection semantics belong under charges until a second feature proves the shared contract (`SQ-25`). |
| `src/index.css` | keep | Global stylesheet imported by the Vite entry. |
| `src/lib/actions/types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/lib/queryClient.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/lib/queryDefaults.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/lib/queryKeys.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/main.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/messages.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/router/AppRoutes.tsx` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/store/auth.selectors.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/store/auth.store.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/store/auth.types.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/types/generated.ts` | keep | Generated OpenAPI drift baseline; intentionally generated and not a hand-maintained runtime module. |
| `src/types/index.ts` | keep | Intentional module/public barrel; its reachable surface was checked with Knip and the import graph. |
| `src/utils/animation.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/clientStatus.ts` | misplaced | Client-domain status rules live in top-level shared utils and deep-import the clients API contract. |
| `src/utils/download.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/dropdownMenuUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/errorParsing.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/utils/filters.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/labels.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/paginationUtils.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/utils/paginationUtils.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/passwordSchema.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/random.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/reportingPeriod.test.ts` | keep | Focused Vitest coverage; it is a deliberate test entry and the full 107-test suite passes. |
| `src/utils/reportingPeriod.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/semanticColors.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/toast.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
| `src/utils/utils.ts` | keep but refactor | Runtime-reachable 261-line module exceeds the 250-line structural-review threshold; split only along real responsibilities. |
| `src/utils/validation.ts` | keep | Runtime-reachable from src/main.tsx or its deliberate module boundary; Knip found no file-level deletion evidence. |
