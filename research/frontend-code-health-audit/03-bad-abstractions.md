# Bad abstractions

This report contains **13** abstraction clusters. Repeated `SQ-*` identifiers are canonical cross-references used by the dirty-code and boundary reports; they are counted once in the overall audit.

| ID | Severity / confidence | Decision |
|---|---|---|
| SQ-02 | P1 · Confirmed | Move connected client behavior to the clients feature; make shared combobox/filter UI domain-neutral. |
| SQ-04 | P1 · Confirmed | Split bidirectional tasks/workQueue ownership into a one-way contract and composition boundary. |
| SQ-08 | P2 · Confirmed | Move to clients; delete the unused callback/effect. |
| SQ-09 | P1 · Confirmed placement; backend policy needs verification | Move client lifecycle capability rules to the clients/backend contract. |
| SQ-11 | P1 · Confirmed | Replace the fake native-select facade with an explicit custom-select contract. |
| SQ-12 | P2 · Confirmed | Delete unused configuration branches and feature-specific animation. |
| SQ-13 | P1 · Confirmed | Split page transformations/components and introduce a real page-hook contract. |
| SQ-14 | P1 · Confirmed | Compose dashboard workflows behind one page hook; keep them domain-specific. |
| SQ-16 | P1 · Confirmed | Split the VAT page-hook dump into focused hooks behind the existing composer. |
| SQ-17 | P1 · Confirmed; one effect hazard needs runtime verification | Correct form-input types, extract payload logic, and replace synchronization effects. |
| SQ-26 | P2 · Confirmed | Delete the taxProfile facade; use a clients-owned read hook. |
| SQ-27 | P2 · Confirmed | Split the tax-calendar API monolith by layer. |
| SQ-28 | P2 · Confirmed | Move display policy out of the annual-report API layer. |

## SQ-02 — Connected client search hidden inside shared UI

- **Purpose / responsibility:** `frontend/src/components/shared/client/ClientSearchInput.tsx:33-287` combines a client-record query, debounce/race handling, portal positioning, keyboard listbox behavior, selection mapping, and presentation. `components/ui/filters/types.ts:46-60`, `FilterPanel.tsx:82-168`, and `ClientPickerFilter.tsx:1-52` add a client-domain branch to the generic filter DSL.
- **Consumers:** direct client-input consumers are `components/shared/client/ClientPickerField.tsx`; `components/ui/filters/ClientPickerFilter.tsx`; dashboard `DashboardPage.tsx`; VAT `VatWorkItemsCreateModal.tsx`; annual reports `CreateReportModal.tsx`; notifications `SendNotificationModal.tsx`; charges `ChargesCreateModal.tsx`; signature requests `CreateSignatureRequestModal.tsx`; tasks `TaskSourceSection.tsx`; advanced payments `CreateAdvancePaymentFlow.tsx` and `GenerateScheduleModal.tsx`; binders `BinderReceivePanel.tsx`. The `client-picker` filter definition is consumed by `useAnnualReportsPage.ts:39`, `useVatWorkItemsPage.ts:34`, `useNotificationsPage.ts:97`, `useChargesFilters.ts:24`, `advancedPayments/constants.ts:79`, and `useBindersPage.ts:23-28`.
- **Configuration surface:** controlled query plus selection callback, error/label/placeholder/helper/size props; internal five state values and four refs; generic filter definition adds `idKey`, optional `nameKey`, label, and placeholder.
- **Problems / evidence:** line 8 imports `clientsApi` from the full clients barrel despite the rule that shared code must not import feature APIs/business logic. Lines 109-140 recreate server-state behavior and silently close on errors. The client-specific filter branch makes every generic filter consumer transitively load the connected picker. It anchors a concrete runtime cycle: `ClientSearchInput.tsx:8` → `clients/index.ts:27` → `ClientChargeDetailPage.tsx:3` → `charges/index.ts:4` → `ChargesCreateModal.tsx:4` → `components/shared/client/index.ts:1`. No tests exist under `components/shared/client`.
- **Concrete replacement shape / decision:** keep a pure listbox/combobox primitive accepting `{items, loading, error, value, onQueryChange, onSelect}`; move the React Query search hook and client result mapping under `features/clients`; compose the connected field from that feature. Remove `client-picker` from `FilterFieldDef` and compose a clients-owned field beside generic filters (or a typed field slot, not another domain switch). **Decision: split and move.**
- **Risk / tests:** medium migration risk across 12 direct and six filter consumers. Add debounce/race, keyboard, portal, error, and selected-value tests before migration.

## SQ-04 — tasks/workQueue pretend to be separate while owning each other's contracts and UI

- **Purpose / responsibility:** task create/update contracts use work-queue source types (`tasks/api/contracts.ts:3,52-77`), while the work-queue response schema validates task statuses (`workQueue/api/contracts.ts:4,103-114`) and workQueue opens/edits tasks.
- **Consumers:** workQueue → tasks: `workQueue/api/contracts.ts`, `hooks/useWorkQueuePage.ts`, `hooks/useWorkQueueActions.ts`, `components/workQueueColumns.tsx`, and `pages/WorkQueuePage.tsx`. Tasks → workQueue: `tasks/api/contracts.ts`, `types.ts`, `hooks/useTaskSourcePicker.ts`, `components/list/TaskListColumns.tsx`, `components/form/TaskModal.tsx`, `components/form/TaskSourceSection.tsx`, `utils/taskFilters.ts`, and `constants/pageConstants.ts`.
- **Configuration surface:** shared source enum, task status enum, source IDs/action payloads, work-queue action descriptors, task modal modes, and filters all cross the boundary.
- **Problems / evidence:** ownership is bidirectional and two imports in `useWorkQueuePage.ts:8-9,35-36` require explicit deep-import suppressions. Runtime cycle: `workQueue/api/contracts.ts:4` → `tasks/index.ts:5` → `TaskModal.tsx:9` → `workQueue/index.ts:1` → `workQueue/api/index.ts:1` → `workQueue.api.ts:4` → contracts. Only `tasks/schemas.test.ts` exists; no workQueue/cross-contract test exists.
- **Concrete replacement shape / decision:** establish one direction. Keep task statuses with tasks; place source-domain identity with the source/work-item contract (a small neutral immutable contract only if both domains genuinely own it); compose TaskModal/actions at `WorkQueuePage` rather than from schema/controller modules. **Decision: split contract ownership and replace UI import with composition.**
- **Risk / tests:** medium. Add schema compatibility tests for source/status summaries and topology fixtures before moving modal/action wiring.

## SQ-08 — `useBusinessesForClient` mixes a reusable read with an unused auto-action

- **Purpose / responsibility:** `frontend/src/hooks/useBusinessesForClient.ts:7-31` queries all businesses for a client and optionally auto-selects the sole result.
- **Consumers:** complete set: `charges/hooks/useChargesPage.ts:5,33`; `charges/components/form/ChargeEditModal.tsx:7,49`; `clients/components/detail/ClientBusinessesCard.tsx:18,38`; `documents/hooks/useClientDocumentsTab.ts:4,29`.
- **Configuration surface:** `{clientId, enabled?, onAutoSelect?}` and output `{businesses, isLoading}`.
- **Problems / evidence:** client-domain server state is misplaced in top-level app hooks. Every consumer omits `onAutoSelect`, so lines 25-29 are an unreachable option/effect. The callback also makes a read hook perform a hidden write sensitive to callback identity. No tests exist.
- **Concrete replacement shape / decision:** move a read-only `useBusinessesForClient({clientId, enabled})` into `features/clients/hooks/queries` and expose it intentionally to charges/documents. Delete `onAutoSelect`; perform any future selection explicitly in the consuming workflow. **Decision: move and simplify.**
- **Risk / tests:** low. Query key and result shape remain unchanged; add one enabled/disabled/data mapping hook test if render-hook infrastructure is introduced.

## SQ-09 — Generic utility functions hide client-lifecycle business policy

- **Purpose / responsibility:** `frontend/src/utils/clientStatus.ts:1-4` decides whether a client is closed and whether creation is locked.
- **Consumers:** `vatReports/components/shared/VatActionButtons.tsx:4,16`; `vatReports/components/detail/VatInvoiceTab.tsx:6,18`; `binders/components/drawer/BinderReceivePanel.tsx:11,65`.
- **Configuration surface:** two predicates accepting `ClientStatus | string | null | undefined`; VAT checks only `closed`, binders checks `closed || frozen`.
- **Problems / evidence:** app-wide utils imports a private clients API type and owns domain policy. Accepting arbitrary `string` defeats enum exhaustiveness. There is no capability contract or test proving whether the two workflow rules should differ; a new backend status silently falls through.
- **Concrete replacement shape / decision:** expose enum-typed client lifecycle predicates/capabilities from the clients feature, or consume backend `available_actions` when it already owns the rule. Preserve workflow differences until backend/product confirms equivalence. **Decision: move and make domain-specific; replace with backend capability if available.**
- **Risk / tests:** low for move/type tightening, medium for policy replacement. Add closed/frozen/unknown workflow tests; backend ownership requires coordination.

## SQ-11 — `Select` exposes an implementation contract it cannot honor

- **Purpose / responsibility:** `components/ui/inputs/Select.tsx:11-60` claims `React.SelectHTMLAttributes<HTMLSelectElement>` while delegating to the custom button/listbox `SelectDropdown.tsx:9-305`.
- **Consumers:** 65 JSX instances across 36 modules: `components/ui/filters/FilterPanel.tsx`; advanced payments `CreateAdvancePaymentModal.tsx`, `AdvancePaymentEditableSections.tsx`; annual reports `ScheduleAddForm.tsx`, `FinancialLineFormParts.tsx`, `CreateReportModalParts.tsx`, `TransitionDetailsForm.tsx`; `AuthorityContactFormFields.tsx`; binders `BinderHandoverPanel.tsx`, `BindersPageDialogs.tsx`, `BinderPeriodFields.tsx`, `BinderReceivePanel.tsx`; charges `ChargeEditModal.tsx`, `ChargesCreateModal.tsx`; clients `ClientBusinessesCard.tsx`, `ClientStatusCard.tsx`, `ClientEditFormSections.tsx`, `CreateClientIdentityStep.tsx`, `CreateClientTaxStep.tsx`; `CorrespondenceModal.tsx`; documents `DocumentEditCard.tsx`, `DocumentsUploadCard.tsx`; notifications `SendNotificationModal.tsx`; reports `AdvancePaymentReportView.tsx`, `VatComplianceReportView.tsx`; signature requests `CreateSignatureRequestModal.tsx`; tasks `TaskDetailsFields.tsx`, `TaskSourceSection.tsx`, `ClientTasksTab.tsx`; `TaxCalendarSettingsPage.tsx`; users `UserFormFields.tsx`; VAT `VatClientSummaryPanel.tsx`, `VatInvoiceTab.tsx`, `VatFileModal.tsx`, `VatInvoiceAddForm.tsx`, `VatPeriodSelect.tsx`.
- **Configuration surface:** the entire native select attribute interface plus label/error/three sizes/options/placeholder/menu width/field class; `SelectDropdown` also accepts array values despite single-selection behavior.
- **Problems / evidence:** `Select.tsx:21-37` drops all un-destructured native props and the ref. `CreateSignatureRequestModal.tsx:125-133` passes ignored `required`; RHF registrations in correspondence, authority contacts, client forms, tasks, and annual reports pass a dropped ref. `SelectDropdown.tsx:192-209` fabricates incomplete objects and asserts them to native select change/focus events. `DatePickerInlineSelect.tsx:1-123` duplicates listbox mechanics; jscpd identifies the clone. Only `SelectDropdown.helpers.test.ts` exists—no component/form/keyboard test.
- **Concrete replacement shape / decision:** define an honest custom contract such as `{value: string; onValueChange(value); onBlur?; name?; triggerRef?; ariaRequired?}`; use a native `<select>` where native registration is required or RHF `Controller` for the custom widget. Extract proven listbox mechanics for date picker reuse only after the contract is corrected. **Decision: replace facade, then consolidate mechanics.**
- **Risk / tests:** medium/high due 65 instances. Stage form consumers first and add required/ref/touched/keyboard/portal tests.

## SQ-12 — `StatsCard` is configured for five screens that do not exist

- **Purpose / responsibility:** `components/ui/layout/StatsCard.tsx:9-229` renders a stat card with optional eyebrow, trend, progress, action text, compact layout, selection, loading, and animated count-up.
- **Consumers:** 16 instances in 13 files: `ClientsStatsSection.tsx`; `WorkQueueStatsSection.tsx`; `VatWorkItemsStatsSection.tsx`; `VatClientSummaryStatsSection.tsx`; `ClientAdvancePaymentsStatsSection.tsx`; `AdvancePaymentsStatsSection.tsx`; `AdvancePaymentSummaryStrip.tsx` (four); `TaxSubmissionStats.tsx`; `ChargesStatsSection.tsx`; `TaxCalendarStatsSection.tsx`; `TaxCalendarSettingsStatsSection.tsx`; `BindersStatsSection.tsx`; `AgingReportHeader.tsx`.
- **Configuration surface:** 17 props including five unused feature branches; six tone records include unused progress settings.
- **Problems / evidence:** no consumer passes `eyebrow`, `trend`, `progress`, `actionLabel`, or `compact`; no spread hides use. Their declarations, `formatTrend`, `ProgressBar` import/config, and render branches are dead. Lines 97-135 animate all numbers from zero on every update, although generic surface animation is disallowed by `ui-guidelines.md:41-43`. No tests exist.
- **Concrete replacement shape / decision:** retain `{title,value,description?,icon?,variant?,selected?,onClick?,className?,loading?}`; delete the other branches and render numbers directly. A feature needing animation should own a reduced-motion-aware wrapper. **Decision: delete configuration and move animation.**
- **Risk / tests:** low for unused branches; medium visual risk for count-up removal. Add a small static/loading/interactive render test.

## SQ-13 — Tax-calendar settings has a facade hook and a controller page

- **Purpose / responsibility:** `taxCalendarSettings/hooks/useTaxCalendarSettings.ts:11-47` starts three queries and one mutation; `pages/TaxCalendarSettingsPage.tsx:23-357` performs validation, grouping/sorting, warning translation, columns, permission/error/loading/empty derivation, and rendering.
- **Consumers:** the hook has exactly one consumer, `TaxCalendarSettingsPage.tsx:183`; the page is routed at `router/AppRoutes.tsx:196`.
- **Configuration surface:** hook takes nullable params plus an enabled boolean and exposes four raw TanStack objects; page retains two state values and many derived branches.
- **Problems / evidence:** the abstraction stops at the library boundary and forces the page to understand every query/mutation implementation detail. Pure parsing/grouping and table configuration live in the 361-line page. The canonical status document already marks it pending. No feature tests exist.
- **Concrete replacement shape / decision:** focused query/mutation hooks plus pure `parseYearRange`, `translateWarning`, `groupEntries`, and column modules, composed by `useTaxCalendarSettingsPage()` returning grouped `{status,headerProps,filters,stats,rules,entries,bootstrap}` slots. **Decision: split and replace the facade with a page composer.**
- **Risk / tests:** medium. Test year/range defaults, warning translation, grouping, query enablement, independent errors, and bootstrap invalidation.

## SQ-14 — Dashboard modal orchestration leaks every setter and mutation

- **Purpose / responsibility:** `dashboard/hooks/useDashboardCreateModals.ts:24-154` coordinates charge, VAT, advance-payment, and client create/restore workflows; `DashboardPage.tsx:29-51,124-190` wires them.
- **Consumers:** exactly one, `DashboardPage`; the page separately consumes `useDashboardPage`.
- **Configuration surface:** the modal hook returns active/deleted/client-picker state, two public setters, five raw mutation objects, four submit callbacks, two formatted errors, and close/reset behavior.
- **Problems / evidence:** the page must derive open/loading/error/restore props and contains an inline async adapter at lines 171-174. It calls two orchestration hooks despite the one-page-hook rule, spreading five workflows over a 193-line page. No dashboard tests exist.
- **Concrete replacement shape / decision:** compose create workflows inside `useDashboardPage` and return deliberate `modals.{charge,vat,advancePayment,client,deletedClient}Props` plus `openCreate(kind)`. Keep this dashboard-specific; do not build a generic modal manager. **Decision: inline composition into the page hook and narrow outputs.**
- **Risk / tests:** medium. Cover each success/error/reset/invalidation/navigation path.

## SQ-16 — `useVatWorkItemsPage` centralizes everything without internal seams

- **Purpose / responsibility:** `vatReports/hooks/useVatWorkItemsPage.ts:58-324` owns URL state, permissions, status summary, grouped query, create/action/send-back/delete mutations, columns, navigation, empty state, and modal props.
- **Consumers:** exactly one, `vatReports/pages/VatWorkItemsPage.tsx`.
- **Configuration surface:** six internal state/query/mutation domains and a 70-line grouped return object (`:254-323`).
- **Problems / evidence:** the page is pure only because the hook became a 267-line dump, contrary to the rule that a page hook composes focused hooks. Lines 69-75 hand-roll URLSearchParams/navigation; lines 46-56 duplicate and assert VAT status values; lines 125-210 mix three mutations and error policies. Only `vatHelpers.test.ts` exists.
- **Concrete replacement shape / decision:** keep `useVatWorkItemsPage` as composer, backed by `useVatWorkItemsFilters`, status-summary/group queries, and `useVatWorkItemListActions`/dialogs. Use `useSearchParamFilters` for the create-param cleanup and the canonical status guard. **Decision: split, retain composer.**
- **Risk / tests:** medium. Test deep-link create cleanup, filter mapping, permissions, mutations/invalidation, and delete/send-back dialogs.

## SQ-17 — Binder receive uses submitted-schema types for incomplete form state

- **Purpose / responsibility:** `binders/hooks/useReceiveBinderDrawer.ts:55-316` coordinates a multi-domain binder intake form; `binders/schemas.ts:6-75` defines submitted validation.
- **Consumers:** exactly one hook consumer, `binders/hooks/useBindersPage.ts:11,131`; the result is wired to `BinderReceivePanel`.
- **Configuration surface:** options `{onSuccess?, initialClient?}`; returns RHF form, client shadow state, businesses/reports/binder/profile data, VAT type, pending state, and four handlers.
- **Problems / evidence:** required IDs are initialized/cleared via `undefined as unknown as ...` at lines 29-30, 286, 293, and 295. Three effects (`:95-112,145-173`) reset correlated fields based on async data and form watches; a late businesses response may overwrite entered periods. The mutation (`:183-279`) performs VAT lookup, nested material payload construction, invalidation, navigation, and silently ignores a second lookup failure. Only `BindersColumns.test.tsx` exists.
- **Concrete replacement shape / decision:** introduce an explicit pre-submit input/default type (or Zod input transform) that allows unset IDs; extract pure `buildReceiveBinderPayload`; isolate client resources and period defaults; reset in explicit client/type/business events rather than async synchronization effects. **Decision: split and correct types.**
- **Risk / tests:** medium/high. Test schema defaults, payload per material combination, delayed queries, user edits, duplicate conflict, and VAT navigation notices before effect changes.

## SQ-26 — `taxProfile` is a facade over clients detail/update, and its write half is dead

- **Purpose / responsibility:** `taxProfile/hooks/useTaxProfile.ts:8-40` renames `clientsApi.getById/update` and `clientsQK.detail` as a tax-profile query/mutation; the feature also contains only `index.ts`, three message lines, and four error-message lines.
- **Consumers:** complete set: `advancedPayments/hooks/useAdvanceRateInsights.ts:1,4`; `useGenerateSchedule.ts:3,14`; `useCreateAdvancePayment.ts:4,20`. `useAdvanceRateInsights` returns `updateAdvanceRate`/`isUpdatingRate` at lines 12-13, but its sole consumer `useClientAdvancePaymentsTab.ts:36` destructures only `advancePaymentFrequency` and `advanceRate`. Thus no screen consumes `updateProfile`, `isUpdating`, or the mutation/messages.
- **Configuration surface:** one `clientId`; returns `{profile,isLoading,error,updateProfile,isUpdating}` and asserts `profileData` back to `ClientRecordResponse` at line 35.
- **Problems / evidence:** there is no tax-profile API, contract, key, page, or distinct resource—only a second name for clients detail/update. It creates an additional feature boundary and participates in the 159-module SCC. The unused mutation is roughly half the hook. No taxProfile tests exist.
- **Concrete replacement shape / decision:** expose a clients-owned read hook returning the specific tax fields (`advance_rate`, `advance_payment_frequency`) for the three advanced-payment consumers; remove the unused write path and delete the taxProfile feature directory. Add a real tax-profile domain only if backend/product introduces one. **Decision: delete facade and replace with clients-owned read contract.**
- **Risk / tests:** low/medium. Preserve query key/cache behavior and loading/error mapping; advanced-payment utility/query-key tests do not cover this read, so add one focused hook/consumer test.

## SQ-27 — `taxCalendar.api.ts` combines contracts, labels, keys, endpoints, and HTTP

- **Purpose / responsibility:** `taxCalendar/api/taxCalendar.api.ts:5-113` declares ten response/parameter/domain types, a message-derived UI label map, query keys, inline endpoint strings, and two HTTP methods.
- **Consumers:** API/key consumers: `hooks/useOpenClientTaxCalendarItem.ts`, `useTaxCalendarGroupItems.ts`, `useTaxCalendarGroups.ts`. Type/label consumers: feature `index.ts`; `constants.ts`; `helpers.ts` and `helpers.test.ts`; `utils.ts` and `utils.test.ts`; `useTaxCalendarGroupsPage.ts`; `components/shared/ClientTaxCalendarTab.tsx`, `ClientTaxCalendarList.tsx`; `components/list/TaxCalendarGroupsContent.tsx`, `TaxCalendarGroupsTable.tsx`, `TaxCalendarStatsSection.tsx`. `api/index.ts:1-4` re-exports the monolith.
- **Configuration surface:** every contract and presentation label is reachable through the API barrel; keys accept mutable params objects and endpoints are embedded in calls.
- **Problems / evidence:** it violates the canonical API folder roles and makes presentation/helpers depend on an HTTP module. `api/index.ts:1-3` already contains an explicit TODO admitting the file intermixes “types/keys/calls/labels.” Helpers/utils tests do not cover API calls or keys.
- **Concrete replacement shape / decision:** split to `contracts.ts`, `queryKeys.ts`, `endpoints.ts`, and a calls-only `taxCalendar.api.ts`; move obligation labels to feature constants/messages. Keep `api/index.ts` as explicit named exports. **Decision: split by existing standard, no new abstraction.**
- **Risk / tests:** low import-migration risk. Add query-key and API serialization tests; existing helper/parser tests should remain unchanged.

## SQ-28 — Annual-report API utilities own UI labels, badge tones, and chart colors

- **Purpose / responsibility:** `annualReports/api/utils.ts:1-81` contains status/client/schedule labels, badge variants, transition helper, and season progress presentation stages.
- **Consumers:** `STATUS_LABELS/getStatusLabel/getStatusVariant`: `useAnnualReportsPage.ts`; `ClientYearComparisonModal.tsx`; `SeasonReportsTable.tsx`; `TransitionTargetSelector.tsx`; `TransitionDetailsForm.tsx`; `StatusTransitionPanelRoot.tsx`; `AnnualReportTimelineTab.tsx`; `ReportHistoryTable.tsx`; external `reports/components/AnnualReportStatusTable.tsx`; private deep consumer `timeline/labels.ts`. `getAllowedTransitions`: `useStatusTransitionPanel.ts`. `getClientTypeLabel`: `SeasonReportsTable.tsx`. `getScheduleLabel/SCHEDULE_LABELS`: `AnnexCard.tsx`, `AnnexHeader.tsx`, `utils/annexHelpers.ts`, `AnnualReportTimelineTab.tsx`. `SEASON_PROGRESS_STAGES`: `SeasonProgressBar.tsx`.
- **Configuration surface:** eight exports are re-exported by `annualReports/api/index.ts:7-16`, mixing HTTP contracts and presentation policy in the public API.
- **Problems / evidence:** the API module imports the UI `BadgeVariant` type at line 1 and contains raw UI colors at lines 74-81. Generic string getters cast typed maps to `Record<string,string>`, weakening enum checking. The timeline deep import/suppression demonstrates the ownership/cycle cost. No annual-report unit test covers these maps/helpers.
- **Concrete replacement shape / decision:** move immutable domain labels and typed getters to feature `constants`/`utils`; put badge/progress presentation tokens beside the owning components; keep API index limited to calls, contracts, and query keys. Leave `available_transitions` server-owned but move its trivial accessor to a feature helper or inline it. **Decision: move/split, not generalize.**
- **Risk / tests:** low/medium because labels affect many screens. Add exhaustive enum-map tests and snapshot/SSR checks for status/schedule labels before moving imports.
