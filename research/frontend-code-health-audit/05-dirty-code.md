# Dirty code and state-management problems

The findings below are evidence-backed structural problems, not formatting preferences or file-size findings by themselves. Repeated `SQ-*` IDs cross-reference other category reports and must be counted once globally.

## Confirmed findings

### SQ-01 — Architecture validation is falsely green

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `frontend/scripts/arch-check.mjs:35-69` extracts and resolves imports; `:72-117` checks cycles; `:132-163` enforces feature barrels. `frontend/package.json:17-18,25-26` makes it part of the normal/strict checks.
- **Problem / evidence:** `resolveImport()` returns every non-relative import unchanged at line 59, so mandated `@/` imports never become source-file graph edges and never match absolute feature paths. `extractImports()` also skips the import after either suppression comment at lines 43-49. `npm run arch:check:strict` passes, while a TypeScript-resolved runtime graph finds SCCs of 159, 4, 2, 2, and 2 modules.
- **Consumers:** both architecture scripts and aggregate `check` scripts; effectively every feature relies on this gate's assurance.
- **Action / risk:** resolve modules with `tsconfig.app.json`, parse imports/re-exports with the TS AST, and add fixture tests for aliases, self-barrels, type-only imports, deep imports, and suppressions. Script risk is low; rollout risk is high because it surfaces existing debt. No checker tests currently exist.

### SQ-07 — `ClientSidebar` is a 267-line mixed controller in shared layout

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `components/layout/ClientSidebar/ClientSidebar.tsx:31-267` owns app branding, responsive overlay/focus behavior, client search/grouping, permissions, create navigation, account display, and logout. `useClientSidebarClients.ts:1-65` owns the client query; sibling card/label/grouping modules own client display policy.
- **Consumers:** one external composition point, `router/AppRoutes.tsx:23,118-124`; its five sibling modules are internal consumers. No tests exist in the directory.
- **Problem / evidence:** the component mixes app shell, auth/user behavior, client data, client grouping, and presentation; it imports clients/users feature behavior and a private clients API type. This prevents isolated testing and puts domain state in shared layout.
- **Action / risk:** move the connected client widget/query/cards to clients ownership and keep a generic responsive shell/focus trap in layout only if reusable. Risk medium; browser-test desktop/mobile open/close, focus return, search, grouping, create, client navigation, and logout.

### SQ-11 — Custom `Select` uses unsafe casts and silently drops native/form props

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `components/ui/inputs/Select.tsx:11-60`; `SelectDropdown.tsx:9-26,192-209,226-304`; duplicate `DatePickerInlineSelect.tsx:1-123`.
- **Problem / evidence:** `Select` claims native select attributes but drops the ref and all un-destructured props. `required` is ignored at `CreateSignatureRequestModal.tsx:125-133`; RHF registrations lose refs in correspondence, authority contacts, clients, tasks, and annual reports. `SelectDropdown` fabricates and casts incomplete native events. jscpd finds a clone with the date-picker listbox.
- **Consumers:** 65 instances across 36 modules; the complete module inventory is in [SQ-11 of the abstraction report](./03-bad-abstractions.md#sq-11--select-exposes-an-implementation-contract-it-cannot-honor). Only `SelectDropdown.helpers.test.ts` exists; no component/form test.
- **Action / risk:** replace the fake native API with a value-based custom contract or a real native select, migrate RHF via supported refs/Controller, then share listbox mechanics. Risk medium/high; add required/ref/touched, keyboard, portal, and disabled-option tests.

### SQ-12 — `StatsCard` stores dead variants and runs an unconditional synchronization animation

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `components/ui/layout/StatsCard.tsx:9-229`; unused prop/branch surface at `:13-25,30-79,145-209`; count-up state/effect at `:97-135`.
- **Consumers:** 16 instances in 13 files: clients, workQueue, two VAT stats modules, three advanced-payment modules, taxDashboard, charges, taxCalendar, taxCalendarSettings, binders, and aging-report header. The exact list is in [SQ-12](./03-bad-abstractions.md#sq-12--statscard-is-configured-for-five-screens-that-do-not-exist). No tests exist.
- **Problem / evidence:** no consumer uses `eyebrow`, `trend`, `progress`, `actionLabel`, or `compact`; all associated branches are dead. Every numeric update is copied into state and animated from zero, rather than rendering derived data; generic feature animation contradicts `ui-guidelines.md:41-43`.
- **Action / risk:** delete dead options and render numbers directly; feature-own any justified animation. Unused-branch risk is low; visible animation removal is medium visual risk. Add static/loading/interactive render coverage.

### SQ-13 — Tax-calendar settings page owns parsing, data orchestration, columns, and every state branch

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx:23-164` contains parsers/translators/grouping/columns; `:166-201` state/query orchestration; `:203-357` error/loading/empty/nested tables. `hooks/useTaxCalendarSettings.ts:11-47` returns four raw query/mutation objects.
- **Consumers:** page route `router/AppRoutes.tsx:196`; the hook has only this page consumer. No feature tests.
- **Problem / evidence:** 361 lines combine pure logic, server state, mutation state, access denial, filters, and presentation. The hook is merely a TanStack facade, so the page violates the documented page contract. Canonical refactor status already marks it pending.
- **Action / risk:** extract pure logic and columns/components, then compose focused queries/mutation behind `useTaxCalendarSettingsPage` grouped slots. Medium risk; test parsing, grouping, warning translation, independent errors/loading, and bootstrap invalidation.

### SQ-14 — Dashboard page spreads five mutation workflows across two hooks and JSX

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `dashboard/pages/DashboardPage.tsx:29-51,124-190`; `hooks/useDashboardCreateModals.ts:24-154`.
- **Consumers:** both hooks have one page consumer; the page renders charge, VAT, advance-payment, client-create, and deleted-client flows. No dashboard tests.
- **Problem / evidence:** the page consumes two orchestration hooks and receives raw mutations/setters/picker internals, forcing it to derive loading/error/open props and reset/restore callbacks. Mutation side effects and UI state are split across files without a stable contract.
- **Action / risk:** compose workflows behind the one dashboard page hook and return prewired modal props; do not generalize them. Medium risk; cover all success/error/reset/navigation/invalidation paths.

### SQ-15 — Client details nests a query component inside a page and silently discards its error

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `clients/pages/ClientDetailsPage.tsx:61-90` (`ClientHeaderMissingDocuments`) calls documents API/query; `:92-100` owns local edit state plus separate query/mutation hooks; `:113-177` builds breadcrumbs/header/tab props.
- **Consumers:** routed client detail/tab pages; the nested component has one in-file consumer at line 141. Only `clients/api/queryKeys.test.ts` exists.
- **Problem / evidence:** the nested component fetches directly and returns `null` for both loading and error at line 70. The page has three state/data owners, and line 84 hardcodes a route despite `CLIENT_ROUTES` use elsewhere.
- **Action / risk:** introduce one client-details page hook and an owner-level document-signal hook with explicit status policy; keep header/meta/banner components presentational. Medium risk because preserving or changing silent error behavior needs a product decision and test.

### SQ-16 — VAT list page hook is a 267-line dump

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `vatReports/hooks/useVatWorkItemsPage.ts:58-324`: URL cleanup, filters, status query, grouping, four write flows, permissions, columns, navigation, empty state, and modals.
- **Consumers:** one, `VatWorkItemsPage.tsx`. Only `vatHelpers.test.ts` exists.
- **Problem / evidence:** manual URL mutation at `:69-75`, duplicated/asserted status values at `:46-56`, mutations at `:125-210`, and a 70-line return at `:254-323` make isolated behavior hard to test. The page became clean by moving all complexity into one place rather than composing focused hooks.
- **Action / risk:** split filters, summary/group reads, and list actions/dialogs behind the existing page-hook composer. Medium risk; test deep-link cleanup, filter params, permissions, errors, invalidation, and dialogs.

### SQ-17 — Binder receive hook defeats its schema with double casts and effect-driven form resets

- **Severity / confidence:** P1 · Confirmed; delayed-response overwrite needs runtime verification.
- **Paths / responsibility:** `binders/hooks/useReceiveBinderDrawer.ts:28-40,55-316`; submitted schema `binders/schemas.ts:6-75`.
- **Consumers:** one hook consumer, `useBindersPage.ts:11,131`; returned controller drives `BinderReceivePanel`. Only `BindersColumns.test.tsx` exists.
- **Problem / evidence:** unset required IDs use `undefined as unknown as` at lines 29-30, 286, 293, 295. Effects at `:95-112,145-173` synchronize/reset form fields from async query changes; the mutation at `:183-279` mixes lookup, payload mapping, toast, cache, and navigation, and silently ignores lookup failure at `:264-266`.
- **Action / risk:** model pre-submit values honestly, extract pure payload construction and resource hooks, and use explicit form events rather than async synchronization effects. Medium/high risk; add schema/default, material-combination payload, delayed query, conflict, and notice-navigation tests.

### SQ-18 — Annual-report URL initialization suppresses its real effect dependency

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `annualReports/hooks/useAnnualReportsPage.ts:71-78` writes the backend default tax year when URL year is missing.
- **Consumers:** `AnnualReportsPage` through its sole page hook; no annual-report page/hook tests.
- **Problem / evidence:** the effect calls `setFilter` but omits it and disables `react-hooks/exhaustive-deps`, contrary to the no-suppression production rule. A long comment explains behavior but does not establish callback stability.
- **Action / risk:** stabilize/include the writer or add `initializeMissingFilters` to the existing URL helper; preserve the `all` sentinel. Low/medium loop/rewrite risk; add missing/default/all/reset URL tests.

### SQ-30 — `.prettierignore` excludes the complete reports feature from the formatter gate

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** `frontend/.prettierignore:4` contains the unscoped directory pattern `reports`; `package.json:16,25-26` treats `format:check` as a required gate. `frontend/src/features/reports/**` contains 21 TS/TSX files and 1,121 LOC.
- **Consumers:** all four report routes and every reports API/component/hook/page module; formatter/check scripts are the enforcement consumer.
- **Problem / evidence:** normal `npx prettier --check src/features/reports` reports success because the directory is ignored. Running the same read-only check with `--ignore-path /dev/null` reports formatting violations in 19 of 21 files. Mixed quotes/semicolons are visible throughout the hooks/pages. No product test can compensate for a disabled formatter gate.
- **Action / risk:** remove/narrow the ignore entry and format the feature mechanically in an isolated commit. Behavior change: none; risk low but diff volume is 19 files. Verify `format:check`, typecheck, and tests after formatting.

### SQ-31 — Report navigation state is local-only and error handling differs by report

- **Severity / confidence:** P2 · Confirmed.
- **Paths / responsibility:** local filters/pagination in `reports/hooks/useAdvancePaymentReport.ts:5-14`, `useAgingReport.ts:12-27`, `useAnnualReportStatusReport.ts:7-24`, and `useVatComplianceReport.ts:7-23`. Raw errors are returned by advance-payment and VAT hooks and exposed with `error?.message` in `pages/AdvancePaymentReportView.tsx:49` and `VatComplianceReportView.tsx:132`; aging/annual-status normalize with `getErrorMessage`.
- **Consumers:** each hook has exactly its same-named page consumer; the pages are routed at `router/AppRoutes.tsx:184-187` and exported from `reports/index.ts:3-6`. No report tests exist.
- **Problem / evidence:** year/month/date/page state that should survive refresh is kept only in `useState`, violating `architecture.md:178-179` and `page-structure.md:181-188`. Back/forward/share links lose selected report state. Two screens may show raw transport/backend error messages while two show stable localized copy.
- **Action / risk:** move year/month/date/page to `useSearchParamFilters` with parsing/defaults and make every hook return `error: string | null`. Medium migration risk because URLs become stateful; test invalid params, defaults, back/forward, page reset on filter changes, and normalized errors.

### SQ-32 — Twenty-four explicit default stale-time assignments obscure the actual query policy

- **Severity / confidence:** P3 · Confirmed.
- **Paths / responsibility:** `lib/queryClient.ts:4-9` sets global `staleTime: QUERY_STALE_TIME.default`; 23 call-site overrides repeat that exact value: `hooks/useBusinessesForClient.ts:18`; annual reports `useAnnualReportsPage.ts:103`, `useCreateReport.ts:75`, `ReportHistoryTable.tsx:25`; VAT `useVatWorkItemGroups.ts:11`, `useVatClientSummary.ts:21`, `useVatWorkItemsPage.ts:98`, `useVatGroupItems.ts:10`; invoices `useChargeInvoice.ts:29`; advanced payments `useAdvancePaymentBatchRows.ts:43`; clients `ClientDetailsPage.tsx:66`, `ClientStatusCard.tsx:62,70`, `useIdNumberConflict.ts:17`, `useFirstBusinessId.ts:10`; binders `BinderHandoverPanel.tsx:54`, `useReceiveBinderDrawer.ts:88,118,129,138`, `BinderIntakesSection.tsx:33,61,67`.
- **Consumers:** all production queries use the shared client through `main.tsx:20-37`; intentional `short`, `medium`, `long`, and `static` overrides are not part of this finding.
- **Problem / evidence:** 24 assignments exist in total; the global one is necessary and the other 23 are behaviorally redundant. Repetition makes deliberate policy exceptions harder to see and creates churn if the default changes. No dynamic usage is involved.
- **Action / risk:** retain the QueryClient default and delete the 23 identical per-query assignments/imports. Behavior change: none with the production client. Risk negligible; typecheck and existing query tests are sufficient.

### SQ-33 — Tax-calendar settings “reset” does not restore its initial range

- **Severity / confidence:** P1 · Confirmed.
- **Paths / responsibility:** `taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx:166-181` initializes both `startYear` and `endYear` to the current year; `resetFilters()` at `:190-193` resets them to current year/current year + 1.
- **Consumers:** the one routed settings page; the resulting range controls entries/summary queries and bootstrap payload at `:183-200`. No tests exist.
- **Problem / evidence:** initial render means a one-year range, while the control labeled reset produces a two-year range. A reset action should restore the declared defaults; two independent literals have drifted.
- **Action / risk:** define one `getDefaultYearRange()` and use it for initial state and reset after confirming whether the intended end is current or next year. User-visible behavior changes on either initial load or reset, so migration risk is medium; add an exact default/reset/query-param test.

## Large-file review

Size alone is not a defect. This table records the concrete disposition after inspecting the largest non-generated files, so cohesive contracts/messages are not mislabeled as dirty.

| File | LOC | Disposition and evidence | Existing tests |
|---|---:|---|---|
| `src/types/generated.ts` | 21,619 | **Keep/generated.** OpenAPI output; never hand-refactor. Regenerate from backend contract. | Covered indirectly by typecheck. |
| `components/ui/table/toolbar/RowActions.tsx` | 407 | **Keep.** Distinct accessible menu/link/item/button behaviors justify the module. Remove only obsolete `RowActionButton.danger`, canonical dead finding `DC-18`; do not split for size alone. | No RowActions-specific test. |
| `annualReports/api/contracts.ts` | 377 | **Keep.** Large but cohesive annual-report request/response schemas/types. Split only along real backend sub-resources, not line count. | No direct contract test. |
| `taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx` | 361 | **Refactor — SQ-13/SQ-33.** Page owns logic, queries, columns, states, and drifting defaults. | None. |
| `components/shared/client/ClientSearchInput.tsx` | 331 | **Move/split — SQ-02.** Connected client API behavior and listbox presentation cross shared/feature boundaries. | None. |
| `vatReports/hooks/useVatWorkItemsPage.ts` | 324 | **Split behind composer — SQ-16.** Filters, reads, writes, columns, navigation, and dialogs. | Only `vatHelpers.test.ts`. |
| `binders/hooks/useReceiveBinderDrawer.ts` | 316 | **Split/correct types — SQ-17.** Four queries, effects, payload, mutation, navigation. | Unrelated columns test only. |
| `vatReports/api/contracts.ts` | 306 | **Keep.** Cohesive runtime schemas and generated-contract adaptation; size is domain breadth, not mixed UI behavior. | VAT helper test only; contract tests would help. |
| `components/ui/inputs/SelectDropdown.tsx` | 305 | **Redesign — SQ-11.** Unsafe native facade and duplicated listbox mechanics. | Helper-only test. |
| `binders/hooks/useBindersPage.ts` | 299 | **Keep.** Canonical grouped page-hook composer; already composes the receive/dialog hooks. Improve its children, not another page layer. | `BindersColumns.test.tsx`. |
| `clients/messages.ts` | 297 | **Keep/data-only.** Large localized copy map but no control flow; splitting only for ownership is optional when a subdomain changes. | No message test required. |
| `clients/components/form/ClientEditFormSections.tsx` | 281 | **Keep but split only with form work.** Cohesive client edit sections; no confirmed generic abstraction defect from size alone. | None. |
| `notes/components/NotesCard.tsx` | 277 | **Keep feature widget; candidate local split.** `NoteComposer`/`NoteRow`/container are distinct presentational units, but the widget legitimately owns one notes hook. Split for testability when notes changes, not as standalone churn. | None. |
| `workQueue/hooks/useWorkQueuePage.ts` | 276 | **Keep composer; boundary fix SQ-04.** Grouped page contract is documented; remove tasks deep-import cycle rather than moving logic back to the page. | None. |
| `dashboard/components/panels/SeasonInsightsCarousel.tsx` | 275 | **Keep feature-specific.** Cohesive carousel/visualization; do not generalize its one-off rings/legend. | None. |
| `vatReports/components/list/VatInvoiceTable.tsx` | 274 | **Keep feature presentation.** Table/edit-row complexity is domain-specific; revisit only alongside VAT invoice workflow tests. | No component test. |
| `binders/components/drawer/BinderReceivePanel.tsx` | 273 | **Keep presentational; simplify imports.** Controller complexity belongs to SQ-17; remove root UI barrel per wrapper report. | None. |
| `components/ui/filters/FilterPanel.tsx` | 267 | **Keep generic core, remove client branch — SQ-02.** Search/select/toggle/date behavior has broad reuse; client-domain switch does not. | No FilterPanel interaction test. |
| `components/layout/ClientSidebar/ClientSidebar.tsx` | 267 | **Move/split — SQ-07.** App shell, auth, and client feature logic are combined. | None. |
| `clients/hooks/useClientsPage.ts` | 265 | **Keep composer.** Canonical grouped list-page contract; no confirmed defect from size alone. | Query-key test only. |
| `workQueue/components/workQueueColumns.tsx` | 264 | **Keep feature-local columns; boundary fix SQ-04.** Domain complexity is visible rather than hidden in a generic table builder. | None. |
| `workQueue/hooks/useWorkQueueActions.ts` | 263 | **Keep feature action hook; boundary fix SQ-04.** Multiple action kinds justify a focused hook; add tests before splitting. | None. |
| `utils/utils.ts` | 260 | **Keep for now; partition only by proven concepts.** Existing semantic modules already host pagination/labels/colors; do not mechanically scatter remaining stable helpers. | Utility coverage varies by consumer. |
| `charges/components/detail/ChargeDetailPanel.tsx` | 258 | **Keep connected feature widget.** Its audit/invoice/notification composition is the screen's real responsibility; no generic extraction recommended. | No panel test. |
