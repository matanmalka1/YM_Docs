# Feature boundaries and dependency topology

Canonical rules used here: feature-specific UI/hooks/API/contracts/constants stay under their owner; shared/layout code must not own feature API/business behavior; internal feature modules must never import their root barrel; cross-feature non-component dependencies use deliberate public contracts; pages are route-role components; circular feature dependencies must be removed (`docs/frontend/architecture.md:33-44,81-113`; `docs/frontend/page-structure.md:136-177`).

## Alias-aware runtime graph

The repository's current architecture script does not resolve `@/` aliases (dirty finding `SQ-01`). A separate TypeScript-resolved graph, excluding type-only imports, found these strongly connected components:

| Runtime SCC | Complete membership summary | Confirmed cause |
|---:|---|---|
| 159 modules | clients 29; vatReports 19; annualReports 14; tasks 12; charges 11; binders 10; advancedPayments 9; workQueue 9; notifications 6; businesses/importExport/timeline 5 each; documents/signatureRequests/taxCalendar 4 each; audit/shared-client 3 each; taxProfile/users/UI filters 2 each; one top-level hook | Shared connected client picker, broad barrels exporting APIs plus pages/components, and multiple bidirectional feature relationships (`SQ-02`, `SQ-04`, `SQ-05`, `SQ-26`). |
| 4 modules | `auth/index.ts`, `pages/ForgotPasswordPage.tsx`, `LoginPage.tsx`, `ResetPasswordPage.tsx` | All three pages import their own root barrel; root barrel exports all three pages. |
| 2 modules | `dashboard/index.ts`, `pages/DashboardPage.tsx` | Page self-imports dashboard root; root exports page. |
| 2 modules | `signing/index.ts`, `pages/SigningPage.tsx` | Page self-imports signing root; root exports page. |
| 2 modules | `taxDashboard/index.ts`, `pages/TaxDashboardPage.tsx` | Page self-imports taxDashboard root; root exports page. |

These are runtime import cycles, not speculative ownership diagrams. Tree-shaking may alter emitted chunks, but it does not remove source-level ESM initialization and maintenance coupling.

## SQ-02 — Shared client UI imports and executes the clients feature

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `components/shared/client/ClientSearchInput.tsx:8,33-287` imports `clientsApi` and owns client querying; `components/ui/filters/types.ts:46-60`, `FilterPanel.tsx:10,82-168`, and `ClientPickerFilter.tsx:1-52` embed it in generic UI.
- **All consumers:** direct: `ClientPickerField.tsx`, `ClientPickerFilter.tsx`, dashboard `DashboardPage.tsx`, VAT `VatWorkItemsCreateModal.tsx`, annual reports `CreateReportModal.tsx`, notifications `SendNotificationModal.tsx`, charges `ChargesCreateModal.tsx`, signature requests `CreateSignatureRequestModal.tsx`, tasks `TaskSourceSection.tsx`, advanced payments `CreateAdvancePaymentFlow.tsx` and `GenerateScheduleModal.tsx`, binders `BinderReceivePanel.tsx`. Filter-definition consumers: `useAnnualReportsPage.ts:39`, `useVatWorkItemsPage.ts:34`, `useNotificationsPage.ts:97`, `useChargesFilters.ts:24`, `advancedPayments/constants.ts:79`, `useBindersPage.ts:23-28`.
- **Problem / evidence:** shared code is upstream of features but imports the full clients feature downstream. Concrete shortest cycle: `ClientSearchInput.tsx:8` → `clients/index.ts:27` → `ClientChargeDetailPage.tsx:3` → `charges/index.ts:4` → `ChargesCreateModal.tsx:4` → `components/shared/client/index.ts:1`. The generic filter union now knows client IDs/names. No shared-client tests exist.
- **Action / risk:** move connected query/result mapping to clients and leave a pure shared combobox; remove the client switch from generic FilterPanel and compose an owner field. Medium risk across 12 direct and six filter consumers; add query/listbox/portal/keyboard/error tests.

## SQ-03 — Fourteen same-feature root-barrel imports violate the explicit internal-import rule

- **Severity / confidence:** P1 · Confirmed.
- **Complete consumer list:**
  - `features/annualReports/pages/AnnualReportsPage.tsx:9-15`
  - `features/auth/pages/ForgotPasswordPage.tsx:10`
  - `features/auth/pages/LoginPage.tsx:13`
  - `features/auth/pages/ResetPasswordPage.tsx:11`
  - `features/binders/pages/BindersPage.tsx:7`
  - `features/charges/pages/ChargesPage.tsx:6`
  - `features/clients/pages/ClientDetailsPage.tsx:25`
  - `features/clients/pages/ClientsPage.tsx:8`
  - `features/dashboard/pages/DashboardPage.tsx:13`
  - `features/signatureRequests/components/shared/SignatureRequestsDashboardPanel.tsx:13`
  - `features/signing/pages/SigningPage.tsx:4`
  - `features/taxDashboard/pages/TaxDashboardPage.tsx:4`
  - `features/users/pages/UsersPage.tsx:9`
  - `features/vatReports/pages/VatWorkItemsPage.tsx:4-9`
- **Problem / evidence:** each internal module imports `@/features/<same-feature>` rather than its actual source. Root barrels re-export pages, so auth/dashboard/signing/taxDashboard form direct SCCs; the others feed the 159-module SCC. These are static imports, not dynamic/type-only uses. The existing checker misses them because it does not resolve aliases. No topology test exists.
- **Action / risk:** mechanically replace each import with relative source imports. Behavior change none; risk low. Verify typecheck and an alias-aware self-barrel fixture/check.

## SQ-04 — tasks and workQueue have no dependency direction

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `tasks/api/contracts.ts:3,52-77` uses `WorkQueueSourceType`; `workQueue/api/contracts.ts:4,103-114` uses task statuses. WorkQueue also executes/edits tasks, while TaskModal/source pickers import workQueue labels/types.
- **Complete cross-feature consumers:** workQueue → tasks: `workQueue/api/contracts.ts`, `hooks/useWorkQueuePage.ts`, `hooks/useWorkQueueActions.ts`, `components/workQueueColumns.tsx`, `pages/WorkQueuePage.tsx`. Tasks → workQueue: `tasks/api/contracts.ts`, `types.ts`, `hooks/useTaskSourcePicker.ts`, `components/list/TaskListColumns.tsx`, `components/form/TaskModal.tsx`, `components/form/TaskSourceSection.tsx`, `utils/taskFilters.ts`, `constants/pageConstants.ts`.
- **Problem / evidence:** `useWorkQueuePage.ts:8-9,35-36` carries two deep-import suppressions specifically because public barrels cycle. Runtime proof: `workQueue/api/contracts.ts:4` → `tasks/index.ts:5` → `TaskModal.tsx:9` → `workQueue/index.ts:1` → `api/index.ts:1` → `workQueue.api.ts:4` → contracts. Only `tasks/schemas.test.ts` exists; no workQueue/cross-contract tests.
- **Action / risk:** establish a one-way contract: task status stays task-owned; source identity belongs to the source/work-item contract or a genuinely neutral immutable contract. Compose TaskModal/action UI at WorkQueuePage. Medium risk; add schema and graph tests.

## SQ-05 — Public barrels pull route pages and connected UI into API-only dependencies

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** broad public surfaces `clients/index.ts:1-40`, `businesses/index.ts:1-3`, `binders/index.ts:1-18`, `vatReports/index.ts:1-14`, `charges/index.ts:1-14`. They export API, hooks, connected widgets, and route pages together.
- **Consumers / evidence:** all 159 SCC members are transitive runtime consumers; complete feature counts are in the graph table. Three minimal exact cycles prove distinct ownership faults:
  1. `clients/api/clients.api.ts:2` → `businesses/index.ts:1` → `BusinessDetailsPage.tsx:9` → `useBusinessDetails.ts:2` → `clients/index.ts:2` → clients API.
  2. `vatReports/hooks/useActiveVatBinder.ts:2` → `binders/index.ts:3` → `BinderDetailDrawer.tsx:8` → `BinderIntakesSection.tsx:13` → `vatReports/index.ts:10` → `VatWorkItemFullPanel.tsx:13` → `VatWorkItemMetaStrip.tsx:5` → `useActiveVatBinder`.
  3. `BinderIntakesSection.tsx:12` → `clients/index.ts:17` → `ClientDetailsTabContent.tsx:3` → `ClientDetailsOverviewTab.tsx:21` → `binders/index.ts:3` → `BinderDetailDrawer.tsx:8` → `BinderIntakesSection`.
- **Problem:** a lightweight endpoint/query dependency executes page/component re-exports; `clientsApi` owns business calls but reaches into businesses solely for endpoint strings, while binders and VAT each fetch/render the other's domain. Changes become feature-wide and initialization order is obscured.
- **Action / risk:** break each minimal cycle locally: co-locate endpoint ownership with the caller; move reverse binder/VAT composition to a page/aggregate owner; prevent client API consumers from reaching client detail page exports. Medium/high if combined, medium per cycle. Existing helper/query-key tests do not cover topology; add fixtures.

## SQ-06 — Timeline label formatting depends on five private feature modules and points back through annual reports

- **Severity / confidence:** P2 · High confidence.
- **Paths / responsibility:** `timeline/labels.ts:2-17,19-57` imports audit plus private binder/client/annual-report/charge label maps; `annualReports/hooks/useReportMutations.ts:7,44-49` imports `timelineQK` back for invalidation.
- **Consumers:** `timeline/normalize.ts:4,82,107-108`; `timeline/components/TimelineMetadata.tsx:8,59,89`; annual report mutation consumes `timelineQK` at line 48.
- **Problem / evidence:** five imports are preceded by `no-restricted-imports` suppressions whose comments explicitly cite barrel cycles. This is a narrowly permitted exception, not itself a rule violation, but it proves the label adapter spans four private owners while annualReports has a reverse dependency. The checker intentionally removes all five edges. `timeline/normalize.test.ts` covers normalization, not every map/direction.
- **Action / risk:** remove the reverse invalidation ownership or inject it at composition, then use deliberate public immutable label contracts. Use a neutral registry only if audit/timeline truly share the same domain contract. Medium risk to labels/cache refresh; add exhaustive map and mutation invalidation tests.

## SQ-07 — Client navigation is stored under shared layout instead of the clients feature

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `components/layout/ClientSidebar/ClientSidebar.tsx:1-267`; `useClientSidebarClients.ts:1-65`; card/labels/grouping siblings.
- **Consumers:** only `router/AppRoutes.tsx:23,118-124` externally.
- **Problem / evidence:** layout imports clients/users behavior (`ClientSidebar.tsx:13,15`), client API/query data (`useClientSidebarClients.ts:3`), and a forbidden private API type (`:4`). The component also owns auth/logout. The clients feature therefore does not own its global client widget. No tests exist.
- **Action / risk:** move the connected widget/query/card/grouping into clients and export it; leave only generic responsive/focus shell in layout if reusable, and separate account footer composition. Medium risk; browser-test desktop/mobile/focus/search/navigation/logout.

## SQ-08 — Client-business reference data is owned by top-level hooks

- **Severity / confidence:** P2 · Confirmed.
- **Path / responsibility:** `hooks/useBusinessesForClient.ts:7-31` imports clients API/query/types and optionally performs selection.
- **Consumers:** `charges/hooks/useChargesPage.ts:5,33`; `charges/components/form/ChargeEditModal.tsx:7,49`; `clients/components/detail/ClientBusinessesCard.tsx:18,38`; `documents/hooks/useClientDocumentsTab.ts:4,29`.
- **Problem / evidence:** a client-domain query sits in app-wide infrastructure. No consumer passes `onAutoSelect`, so its effect is also dead. No tests exist.
- **Action / risk:** move a read-only query hook under clients and expose it publicly; delete the callback/effect. Low risk.

## SQ-09 — Client lifecycle policy is owned by app-wide utils

- **Severity / confidence:** P1 · Confirmed placement; backend ownership needs verification.
- **Path / responsibility:** `utils/clientStatus.ts:1-4` imports a private clients API type and decides action locks.
- **Consumers:** VAT `VatActionButtons.tsx:4,16`, `VatInvoiceTab.tsx:6,18`; binders `BinderReceivePanel.tsx:11,65`.
- **Problem / evidence:** generic utils contains client-specific policy and accepts arbitrary strings, while VAT and binder workflows use different predicates without an authoritative capability contract. No tests exist.
- **Action / risk:** move typed lifecycle helpers to clients or replace them with backend `available_actions` after coordination. Low risk for ownership move, medium for policy change; test closed/frozen/unknown states.

## SQ-10 — A reusable advanced-payment tab and its components are under `pages/`

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `advancedPayments/pages/ClientAdvancePaymentsTab/ClientAdvancePaymentsTab.tsx`, `ClientAdvancePaymentsCards.tsx`, `ClientAdvancePaymentsHeader.tsx`, `ClientAdvancePaymentsStatsSection.tsx`.
- **Consumers:** the tab is exported by `advancedPayments/index.ts:2` and consumed only by `clients/components/detail/ClientDetailsOverviewTab.tsx:20,105`; its three children have only the tab as consumer. No router uses any of the four as a route. No render tests exist (only advanced-payment query-key/utility tests).
- **Problem / evidence:** route inventory treats non-route widgets as pages and the public feature boundary points into a page subfolder for a reusable detail component.
- **Action / risk:** move all four to `advancedPayments/components/detail` (or shared only if a second internal area uses them). Path-only, low risk.

## SQ-26 — `taxProfile` creates a feature boundary for a resource owned entirely by clients

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `taxProfile/hooks/useTaxProfile.ts:8-40` wraps `clientsApi.getById/update` and `clientsQK.detail`; `taxProfile/index.ts:1-2`, messages, and errors complete the otherwise empty feature.
- **Consumers:** only advanced-payment hooks `useAdvanceRateInsights.ts:1,4`, `useGenerateSchedule.ts:3,14`, and `useCreateAdvancePayment.ts:4,20`. The sole `useAdvanceRateInsights` consumer reads only rate/frequency, making all taxProfile mutation outputs dead.
- **Problem / evidence:** no tax-profile endpoint, contract, query key, page, or distinct backend resource exists. The facade adds clients → taxProfile → advancedPayments coupling and two taxProfile modules to the 159 SCC; line 35 adds a response assertion. No tests exist.
- **Action / risk:** replace with a clients-owned focused read hook and delete the taxProfile feature/write facade. Low/medium risk; preserve clients cache key and add one advanced-payment profile-read test.

## SQ-27 — Tax-calendar presentation and contracts depend on an HTTP monolith

- **Severity / confidence:** P2 · Confirmed.
- **Path / responsibility:** `taxCalendar/api/taxCalendar.api.ts:5-113` mixes contracts, message-derived labels, query keys, inline endpoints, and calls; `api/index.ts:1-4` explicitly acknowledges the debt.
- **Consumers:** API/key: `useOpenClientTaxCalendarItem.ts`, `useTaxCalendarGroupItems.ts`, `useTaxCalendarGroups.ts`. Types/labels: feature `index.ts`, `constants.ts`, `helpers.ts`/test, `utils.ts`/test, `useTaxCalendarGroupsPage.ts`, `ClientTaxCalendarTab.tsx`, `ClientTaxCalendarList.tsx`, `TaxCalendarGroupsContent.tsx`, `TaxCalendarGroupsTable.tsx`, `TaxCalendarStatsSection.tsx`.
- **Problem / evidence:** nearly every feature layer imports the HTTP module because it also owns types/labels. This violates the canonical API layout and makes UI policy reachable through API barrels. Existing tests cover helpers/parsers, not calls/keys.
- **Action / risk:** split contracts/queryKeys/endpoints/calls and move labels to feature constants/messages. Low import-migration risk; add key/serialization tests.

## SQ-28 — Annual-report API surface exports presentation policy and causes private cross-feature imports

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `annualReports/api/utils.ts:1-81`; API re-export `api/index.ts:7-16`.
- **Consumers:** status maps/getters/variants: `useAnnualReportsPage.ts`, `ClientYearComparisonModal.tsx`, `SeasonReportsTable.tsx`, three status-transition components, `AnnualReportTimelineTab.tsx`, `ReportHistoryTable.tsx`, external `reports/AnnualReportStatusTable.tsx`, and private `timeline/labels.ts`. Other exports: `getAllowedTransitions` → `useStatusTransitionPanel.ts`; client-type label → `SeasonReportsTable.tsx`; schedule labels/getter → `AnnexCard.tsx`, `AnnexHeader.tsx`, `annexHelpers.ts`, `AnnualReportTimelineTab.tsx`; progress stages → `SeasonProgressBar.tsx`.
- **Problem / evidence:** API code imports UI `BadgeVariant`, contains display strings and Tailwind colors, and weakens typed maps through string casts. Timeline requires a suppressed private API-utils import because the public barrel cycles. No map/helper tests exist.
- **Action / risk:** move immutable labels/domain helpers to feature constants/utils and visual tones/stages beside UI; leave API index to calls/contracts/keys. Low/medium risk across many labels; add exhaustive enum map and render tests.

## SQ-34 — Timeline retains a removed future/deadline contract that no producer can satisfy

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `timeline/normalize.ts:7` still includes `'future'` in `TimelineFilterKey`; `:140-145` always returns `upcomingDeadlines: []`; `timeline/lib/timelineGroups.ts:21-33` contains a default-open branch for events with the future key.
- **All consumers:** `normalizeTimelineEvents` is consumed by the client timeline page flow and `normalize.test.ts`; no caller destructures or reads `upcomingDeadlines`. `getDefaultOpenTimelineGroups` is consumed by timeline grouping UI, but every current event key is built from `FILTER_BY_EVENT_TYPE` at `normalize.ts:19-38`, whose entries emit only `past` plus domain keys—never `future`.
- **Problem / evidence:** the `future` side of the group condition cannot execute, and the returned empty deadlines collection has no consumer. Git-visible history (`8785ea17`) removed the `taxDeadlines` source while deliberately leaving this empty compatibility shape. It now suggests that timeline owns upcoming tax obligations even though that behavior moved to tax-calendar surfaces.
- **Action / risk:** remove `'future'`, `upcomingDeadlines`, and the unreachable default-open check; keep timeline historical-event ownership explicit. If future obligations are reintroduced, compose a real tax-calendar query rather than preserving an empty adapter. Low risk; update `timeline/normalize.test.ts` and add one default-group test.

## Recommended boundary repair order

1. Repair `arch-check` alias resolution (`SQ-01`) and run it report-only.
2. Remove the 14 self-root imports (`SQ-03`).
3. Move/delete the smallest misplaced owners: `useBusinessesForClient`, client status utils, advanced-payment tab, taxProfile (`SQ-08/09/10/26`).
4. Separate client connected UI/sidebar from shared/layout (`SQ-02/07`).
5. Split API/display ownership (`SQ-27/28`).
6. Break tasks/workQueue and the three minimal broad-barrel cycles (`SQ-04/05`).
7. Delete the stale future/deadline compatibility shape (`SQ-34`).
8. Remove timeline suppressions after dependency direction is one-way (`SQ-06`), then make the repaired graph check blocking.
