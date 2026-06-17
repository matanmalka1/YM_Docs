## Scope

This file owns only:
- A per-wave changelog of the Page Architecture Convergence effort (what code changed and why).

This file must not contain:
- Per-page refactor status (that lives in `page-refactor-status.md`).
- The architectural standard itself (that lives in `page-structure.md`).

Source of truth: reference

---

# Page Architecture Convergence — Change Log

Goal: converge ~20 feature list pages onto one canonical page shape so a page is
written the same way every time. Reference-first, additive-only, no big-bang.
Plan reference: `.claude/plans/context-ym-tax-composed-galaxy.md`.

Related docs:
- `page-refactor-status.md` — per-page status table.
- `page-structure.md` — the standard pages converge toward.

---

## The grouped page-hook contract

Each `useXPage` returns named, grouped slots (naming/location convention — no
shared type yet):

- `status` — `{ isLoading; isFetching; error: string | null; loadingMessage? }`
- `headerProps` — `{ title; description?; actions? }` (data only; page renders `<PageHeader />`)
- `stats` — feature-owned stats props (one slot; retire counters/summary/workflowStats sprawl)
- `filters` — feature-owned filters, incl. `resetFilters()` and optional `hasActiveFilters`
- `table` — `{ data; columns; pagination?; emptyState? }`
  - page renders `page.table.data` / `page.table.columns`, never `page.<entityPlural>`
  - columns prepared in the hook/feature helper, never built in page JSX
  - `emptyState` is an `EmptyStateConfig` (`isEmpty`, `isFiltered`, copy, action)
- `modals` / `drawers` — slot location locked; internal shape converges by example
- `permissions` — capability booleans grouped under one name

Rules: hooks return config (not JSX); hooks compose smaller feature hooks rather
than becoming dumping grounds; pages own no queries/URL parsing/columns/modal
state/error conversion/empty derivation.

### Sanctioned reference-page exceptions (read before copying for Wave 1+)

The Binders reference is fully converged EXCEPT one deliberate, approved item — do
NOT treat it as a pattern to replicate or "fix":

- **Header action button is page-composed JSX.** The header "קליטת חומר" button
  lives in `BindersPage` (label + icon), wired to a hook callback
  (`drawers.openReceive`). `PageHeader.actions` takes a `ReactNode` today, and a
  `headerProps.actions` config array + config→button mapper is NOT worth building
  for a single button. Deferred to Wave 8 (reassess-extraction). When a later page
  also needs header actions, that's the signal to revisit — not before.

This is the only exception. Everything else (status/stats/filters/table/modals/
drawers, columns, empty copy, error conversion) is hook-owned.

---

## Wave 0A — Binders reference page (contract-only)

Status: **done.** Automated verification + Atlas browser smoke passed. Modal/drawer
ownership intentionally deferred to Wave 0B.

Smoke-test note: Atlas flagged the detail-drawer "מוכן למסירה" button executing
immediately with no confirmation. Verified via `git diff` that Wave 0A preserved
this exactly — it is pre-existing behavior on `main` (that button maps to the
immediate `onMarkReadyForHandover`; the bulk button is the one with a dialog).
Not a refactor regression. A confirmation dialog was subsequently added as a
separate behavior change (see "Mark-ready-for-handover confirmation" below).

Files changed:
- `features/binders/hooks/useBindersPage.ts`
- `features/binders/pages/BindersPage.tsx`

Changes:
1. `useBindersPage` now returns the grouped contract
   (`status` / `headerProps` / `stats` / `filters` / `table` / `modals` / `drawers`).
2. Column construction (`buildBindersColumns`) moved out of the page into the hook;
   the page no longer imports it.
3. `useBindersPageDialogs` composed inside `useBindersPage` (so column action
   callbacks can reference dialog handlers). Dialog/selection **state ownership
   unchanged** — still in `useBindersPageDialogs` / `useBinderSelection`; only the
   call site relocated, grouped under `modals` / `drawers`.
4. `table.emptyState` is an `EmptyStateConfig`; loading/error/empty derivation moved
   into the hook (`status.error: string | null`).
5. `BindersPage` reduced to slot composition (`const page = useBindersPage(...)`),
   206 → 156 lines. Receive drawer (`receiveOpen`) stays page-owned per the 0A
   boundary; hook receives `onOpenReceive` so it still owns empty-state action copy.

Behavior-preserving side-effects:
- Wrapped `pageItems` in `useMemo` — moving the columns memo into the hook surfaced
  an unstable-ref lint warning the old page-level memo had masked.

Verification: `npm run typecheck` (zero binder errors) · `npm run lint` (binder
files clean) · `npm run arch:check` (no violations) · `npm run build` (green).
No binder unit tests exist.

---

## Mark-ready-for-handover confirmation (behavior change, not part of 0A)

Triggered by the Wave 0A smoke-test (above). The non-bulk "מוכן למסירה" action
fired immediately with no confirmation — a pre-existing gap, now closed. This is a
**behavior change**, kept separate from the refactor.

Files changed:
- `features/binders/hooks/useBinderMutations.ts` — expose
  `isMarkingReadyForHandover` (mirrors `isDeleting`).
- `features/binders/hooks/useBindersPageDialogs.ts` — new `confirmReadyForHandover`
  dialog state/handlers (`openReadyForHandoverDialog` / `closeReadyForHandoverDialog`
  / `confirmReadyForHandover`); takes `markReadyForHandover` as a param.
- `features/binders/hooks/useBindersPage.ts` — row + detail-drawer
  `onMarkReadyForHandover` now open the dialog instead of mutating immediately;
  `isMarkingReadyForHandover` exposed under `modals`.
- `features/binders/components/dialogs/BindersPageDialogs.tsx` — renders a
  `ConfirmDialog` ("האם לסמן את קלסר … כמוכן למסירה?"), normal (non-danger) variant
  since the status is reversible.

Behavior: both entry points (table row action + detail drawer) now require
confirmation. The bulk variant was already gated by its own dialog — unchanged.

Verification: lint + typecheck (binder-clean) · arch:check (no violations) ·
`npm run build` (green).

---

## Wave 0B — Binders orchestration cleanup

Status: **done.** Behavior identical to post-0A; automated verification passed.
`BindersPage` is now pure slot composition (75 lines, no `useState`, no inline
handler expressions, no copy beyond the one page-composed header action button).

Files changed:
- `features/binders/hooks/useBindersPage.ts`
- `features/binders/pages/BindersPage.tsx`

Changes:
1. **Receive-drawer ownership → hook.** `useState(receiveOpen)` +
   `useReceiveBinderDrawer(...)` moved out of the page into `useBindersPage`.
   The hook exposes `drawers.openReceive` (used by the header button and the
   empty-state action) and a pre-wired `drawers.receive` (open, onClose-with-reset,
   plus all `ReceiveBinderDrawer` form props). The `onOpenReceive` param was
   dropped — `useBindersPage()` now takes no arguments.
2. **`drawers.detail` pre-wired in the hook.** The 9 conditional
   `selectedBinder ? () => … : undefined` handler expressions moved out of the page;
   the page renders `<BinderDetailDrawer {...drawers.detail} />`.
3. **`modals.dialogsProps` pre-wired in the hook.** The full `BindersPageDialogs`
   prop set (incl. `getBinderNumberLabel` pre-bound to `pageItems` + `selectedBinder`)
   is assembled in the hook; the page renders `<BindersPageDialogs {...modals.dialogsProps} />`.
   `useBindersPageDialogs` remains the internal state owner — no longer exposed raw.
4. Table row-click moved to `table.onRowClick` (was `drawers.onSelect`).

Ownership note: dialog / selection / receive **state** still lives in
`useBindersPageDialogs` / `useBinderSelection` / `useReceiveBinderDrawer`. The hook
only assembles ready-to-spread prop objects (config, not JSX), so the page composes
JSX from slots.

Accepted exception: the header "קליטת חומר" action button stays page-composed
(JSX + label), wired to `drawers.openReceive`. Converting header actions to a
config array (`headerProps.actions`) is deferred to Wave 8 — not worth a
config→button mapper for a single button, and `PageHeader` takes `actions` as a
ReactNode today.

Verification: `npm run typecheck` (binder-clean) · `npm run lint` (binder files
clean) · `npm run arch:check` (no violations) · `npm run build` (green). No binder
unit tests exist. Behavior-preserving → diff review vs post-0A; live re-smoke
optional.

---

## Wave 1 — UsersPage

Status: **done.** Behavior identical to `main` (one additive item, see below);
automated verification passed. `UsersPage` is now pure slot composition (~80
lines, no `useState`, no `useMemo`, no `useAuthStore`, no `buildUserColumns`
import, no inline handler expressions) — except the two page-composed header
action buttons and the non-advisor permission branch (both sanctioned).

Files changed:
- `features/users/hooks/useUsersPage.ts`
- `features/users/pages/UsersPage.tsx`
- `features/users/components/UsersFiltersBar.tsx`

Changes:
1. **Grouped contract.** `useUsersPage` now returns
   `status` / `headerProps` / `filters` / `table` / `modals` / `permissions`
   (no `stats` — Users has none). The old flat object
   (`users`/`loading`/`error`/modal setters/actions/`isAdvisor`) is gone.
2. **Columns into the hook.** `buildUserColumns` is now built in a `useMemo`
   inside `useUsersPage` (it reads `currentUserId` via `useAuthStore` hook-side);
   `table.columns` is consumed by the page, which no longer imports
   `buildUserColumns` or calls `useAuthStore`.
3. **Toggle-active dialog state into the hook.** `pendingToggle` `useState` +
   confirm/cancel moved out of the page; the hook exposes a pre-wired
   `modals.toggleActiveProps` (open/title/message/labels/`isLoading`/onConfirm/
   onCancel). Title/message/labels still derive from `pendingToggle.is_active`;
   confirm toggles + closes, cancel closes. The page renders
   `<ConfirmDialog {...modals.toggleActiveProps} />`.
4. **Modal prop objects pre-wired.** `createProps` / `editProps` /
   `resetPasswordProps` / `auditLogsProps` + `openCreate` / `openAuditLogs`
   openers assembled in the hook; the page spreads them. Modal **state** stays
   owned by `useUsersPage`'s `useState` — only the call site moved into config.
5. **`status` group.** `status.isFetching` added (was absent);
   loading/error/`loadingMessage` ('טוען משתמשים...') derived hook-side
   (`status.error: string | null`).
6. **`table.emptyState`.** Now an `EmptyStateConfig`-shaped object
   (`isEmpty: users.length === 0`, `isFiltered: Boolean(search || is_active)`,
   title/message/action) mirroring Binders; the page reads title/message/action.
7. **Page-size preserved.** `table.pagination.onPageSizeChange` wraps
   `handleFilterChange('page_size', String(size))`.

Additive item (approved, EC-5): **`filters.resetFilters`.**
`UsersFiltersBar` already rendered a reset control (via `FilterPanel.onReset` →
`ActiveFilterBadges`), so **no new UI button was added**. The reset logic moved
from an inline two-call closure in `UsersFiltersBar` into the hook
(`resetFilters = () => setFilters({ search: '', is_active: '' })` — clears
search + is_active, resets to page 1, preserves `page_size`, identical to prior
behavior). `UsersFiltersBar` now takes an `onReset` prop wired to
`filters.resetFilters`.

Sanctioned exceptions kept page-side (per reference rules):
- The two header action buttons (לוג ביקורת, משתמש חדש) stay page-composed JSX,
  wired to `modals.openAuditLogs` / `modals.openCreate` (same header-action
  exception as Binders; no config→button mapper).
- The non-advisor permission branch (EC-1) stays a page-level early return:
  simple `PageHeader` + warning `Alert`, no action buttons. Its copy
  (description "ניהול חשבונות משתמשים במערכת") differs from the advisor
  `headerProps.description` and is preserved exactly. Page reads
  `permissions.isAdvisor`.

Behavior-preserving notes:
- Self-row (EC-2): `currentUserId` is now read in the hook and passed to
  `buildUserColumns` unchanged — `UserRowActions` still disables self-actions.
- `is_active` guard (EC-3) unchanged — only 'true'/'false' valid, else undefined.

Verification: `npm run typecheck` · `npm run lint` (`--max-warnings=0`) ·
`npm run arch:check` (no violations) · `npm run build` (green; pre-existing
large-chunk warning only). No users unit tests exist, and the repo has no
`renderHook` test infra — none added (setting up a QueryClient + Router hook
harness from scratch is out of scope for a behavior-preserving refactor).
Behavior-preserving → diff review vs `main`; optional Atlas smoke of
`/settings/users`.

---

## Wave 2 — ChargesPage

Status: **done.** Behavior identical to `main`; automated verification passed.
`ChargesPage` is now pure slot composition (~80 lines, no `useState`,
no `useEffect`, no `useMemo`, no `useSearchParamFilters`, no `buildChargeColumns`
import, no inline handler expressions) — except the one page-composed advisor-gated
header button (sanctioned header-action exception).

Files changed:
- `features/charges/hooks/useChargesPage.ts`
- `features/charges/pages/ChargesPage.tsx`

Changes:
1. **Grouped contract.** `useChargesPage` now returns
   `status` / `headerProps` / `stats` / `filters` / `table` / `drawers` /
   `modals` / `permissions`. The old flat object
   (`charges`/`loading`/`error`/`setFilter`/`setPage`/`setSearchParams`/modal
   setters/actions/`isAdvisor`) is gone. `status.isFetching` added (was absent);
   `status.error: string | null` derived hook-side; `loadingMessage`
   ('טוען חיובים...') moved into the hook.
2. **Columns + `allIds` into the hook.** `buildChargeColumns` + `allIds` `useMemo`
   moved out of the page; `table.columns` is consumed by the page, which no longer
   imports `buildChargeColumns`. Column action callbacks (`onOpenDetail`,
   `onSendNotification`) now reference hook-owned state/openers.
3. **Selected-charge + modal state into the hook.** `selectedChargeId`,
   `showCreateModal`, and `notificationContext` `useState`s moved out of the page.
   The hook exposes `drawers.detail` (`chargeId`/`onClose`),
   `modals.createProps` (open/createError/createLoading/onClose/onSubmit), and
   `modals.notificationProps` (null when closed; pre-wired
   open/onClose/clientRecordId/initialTrigger/entityId/disableTriggerChange when a
   notification context exists). `openCreate`/`openNotification` openers are
   hook-owned `useCallback`s; the notification opener is wired into the columns.
4. **URL-param effects into the hook.** Both effects moved: the `charge_id`
   deep-link (positive-int guard → opens detail drawer) and the `create=1`
   deep-link (advisor-only → opens create modal then strips `create` via
   `setSearchParams(next, { replace: true })`). `useSearchParamFilters` is now
   internal to the hook — the page imports it not at all.
5. **`filters` group.** `filters.values` / `filters.onFilterChange` (= `setFilter`)
   / `filters.resetFilters`. The page's old inline clear
   (`setSearchParams(new URLSearchParams())`) converged to `resetFilters` (now
   wired to `ChargesFiltersCard.onClear`). Behavior identical — both clear all
   params; `resetFilters` additionally sets `page=1` explicitly, which the prior
   form achieved via the `page` default.
6. **`stats` group.** `{ stats, isAdvisor, currentStatus: filters.status,
   onStatusClick }` feeds `ChargesSummaryBar`; `onStatusClick` is the
   `setFilter('status', …)` callback (toggle logic still lives in the bar).
7. **`permissions.isAdvisor`.** Grouped under `permissions`; `isSecretary` stays
   internal to the hook (feeds `canAct = isAdvisor || isSecretary`).

Table slot shape (new this wave — see checkpoint): `table` holds
`data` / `columns` / `pagination` / `selection` / `onOpenCharge` / `onCreateCharge`.
`selection` (`selectedCount`/`bulkLoading`/`onBulkAction`/`onClearSelection`) is a
nested sub-group inside `table`, NOT a separate top-level slot (see checkpoint
decision). The page spreads these into the existing `ChargesTableBlock` (composite
bulk-toolbar + `PaginatedDataTable` wrapper) — its flat prop interface was left
unchanged (smallest change).

Edge cases preserved: EC-1 (create=1 advisor-only + replace-strip),
EC-2 (`charge_id` deep-link + `closeChargeDetail` clears via
`setFilter('charge_id', '', false)` — page not reset), EC-3 (bulk selection set
drives column checkboxes + toolbar), EC-4 (advisor gates header button +
`submitCreate`), EC-5 (cross-feature `SendNotificationModal` with
`disableTriggerChange`/`entityId`/`clientRecordId`), EC-6 (empty state stays
component-owned in `ChargesTableBlock` — see checkpoint).

Behavior-preserving side-effect:
- Wrapped `chargeItems` in `useMemo` — moving the columns memo into the hook
  surfaced the same unstable-ref lint warning Binders hit (the old page-level memo
  masked it).

Verification: `npm run typecheck` · `npm run lint` (`--max-warnings=0`) ·
`npm run arch:check` (no violations) · `npm run build` (green; pre-existing
large-chunk warning only) · `npm run test` (42 passed / 11 files). No charges unit
tests exist, and the repo still has no `renderHook` / Router+QueryClient hook
harness — focused URL-state tests were not cheap to add (same finding as Wave 1)
and were not added. Behavior-preserving → diff review vs `main`; optional Atlas
smoke of `/charges`.

---

## CONTRACT CHECKPOINT (after Wave 2)

Three real migrations now exist: Binders (reference), Users, Charges. Charges was
the first to introduce a composite table wrapper (`ChargesTableBlock` = bulk
toolbar + `PaginatedDataTable` + component-owned empty state) instead of a raw
`PaginatedDataTable`, plus a bulk-selection concept and a cross-feature modal.
Decisions:

1. **Does `table` hold selection + composite-wrapper props cleanly, or does bulk
   selection deserve its own slot?** — **Keep selection nested under `table`.**
   Bulk selection is table-scoped (the checkbox column lives in the table columns;
   the toolbar renders directly above the same table). Promoting it to a top-level
   `selection` slot would split one cohesive concern across two slots and imply a
   reusable cross-page selection contract we don't have. `table.selection`
   (`selectedCount`/`bulkLoading`/`onBulkAction`/`onClearSelection`) reads cleanly
   and keeps the contract's top level stable. Contract addition: `table` MAY carry
   an optional feature-owned `selection` sub-group when a page has bulk actions.

2. **Is `table.emptyState` still the right home when a wrapper owns empty
   rendering?** — **No change forced; component-owned empty state is acceptable
   when a composite wrapper already owns it.** Binders/Users render a raw
   `PaginatedDataTable`, so the hook must supply `table.emptyState`
   (`EmptyStateConfig`) — the page would otherwise derive empty copy. Charges
   renders `ChargesTableBlock`, which already owns empty derivation internally
   (`getChargesEmptyState`, gated on `isAdvisor`); the page never touches empty
   copy regardless. Lifting it to `table.emptyState` would move copy out of the
   wrapper only to thread it back in — no DoD benefit, and it would change which
   layer owns the advisor-conditional message. Contract clarification:
   `table.emptyState` is **required when the page renders a raw table**, and
   **optional when a composite table wrapper already owns empty rendering** (the
   DoD "page must not derive empty state" is satisfied either way). Empty
   derivation must never live in the page.

3. **Any contract revision needed before Wave 3+?** — **The contract holds.** Only
   the two clarifications above (both additive, neither breaks Binders/Users):
   - `table.selection?` optional sub-group for bulk-action pages.
   - `table.emptyState` optional when a composite wrapper owns empty rendering.
   No top-level slot changes. Wave 3 (`ClientsPage`) proceeds on the current shape;
   it renders a raw table, so it must supply `table.emptyState` (preserve its
   richer empty state) and has no bulk selection (no `table.selection`).

---

## Wave 3 — ClientsPage

Status: **done.** Behavior identical to `main`; automated verification passed.
`ClientsPage` is now pure slot composition (~88 lines, no `useState`,
no `useEffect`, no `useNavigate`/`useSearchParams`, no `buildClientColumns`/
`useClientQuery` import, no empty-state derivation, no inline handler
expressions) — except the two page-composed header action buttons and the
view-only `Alert` (both sanctioned).

Files changed:
- `features/clients/hooks/useClientsPage.ts`
- `features/clients/pages/ClientsPage.tsx`
- `features/clients/components/list/ClientsStatsSection.tsx` (new)
- `features/clients/components/edit/ClientEditDrawer.tsx` (new)
- `features/clients/index.ts` (barrel exports for the two new components)

Two sanctioned new components (the only ones for this wave):
1. **`ClientsStatsSection`** — mirrors `BindersStatsSection`; replaces the inline
   3× `StatsCard` grid (פעילים / מוקפאים / סגורים). Driven by the `stats` slot
   (`values` / `selected` / `onStatusClick`). The status toggle
   (`status === x ? '' : x`) lives in the hook's `onStatusClick`.
2. **`ClientEditDrawer`** — `DetailDrawer` + `ClientEditForm` + `ModalFormActions`.
   Returns `null` when `!open`. Driven by `drawers.edit`
   (`open` / `onClose` / `client` / `isLoading` / `error` / `onSave` /
   `updateLoading` / `formId`).

Changes:
1. **Grouped contract.** `useClientsPage` now returns
   `status` / `isEmptyState` / `headerProps` / `stats` / `filters` / `table` /
   `drawers` / `modals` / `permissions`. The old flat object
   (`clients`/`error`/`loading`/`setPage`/modal setters/actions/`can`) is gone.
   `status.isFetching` added (was absent); `status.error: string | null` and
   `loadingMessage` ('טוען לקוחות...') moved into the hook.
2. **Columns into the hook.** `buildClientColumns` built in a `useMemo` inside the
   hook (edit action gated by `can.editClients`); the page no longer imports it.
3. **Edit-drawer state + detail query into the hook.** `editingClientId`
   `useState` + the second `useClientQuery({ clientId: editingClientId })` moved
   out of the page; exposed pre-wired as `drawers.edit`. `onSave` updates then
   closes; `EDIT_FORM_ID` ('client-edit-form-list') is hook-owned and threaded
   through `drawers.edit.formId`.
4. **Modal state + create=1 effect + navigation into the hook.**
   `showCreateModal` / `showImportExport` `useState`s, the `?create=1` deep-link
   effect (advisor-only → opens create then strips the param via
   `navigate({ search }, { replace, preventScrollReset })`), and row-click
   navigation moved into the hook. The hook now owns `useNavigate`; the page
   imports neither `useNavigate` nor `useSearchParams`.
5. **Pre-wired modal props.** `modals.createProps` (incl. the deleted-client
   `onRestoreDeletedClient` → restore + navigate detail), `modals.importExportProps`,
   `modals.deletedClientProps` (incl. `onDismiss` → dismiss + reopen create), plus
   `modals.openCreate` / `modals.openImportExport` openers. The create↔deleted-dialog
   interplay (`open: showCreateModal && !deletedClientDialogOpen`) is hook-owned.
6. **`filters` group.** `values` / `onFilterChange` / `resetFilters` (= `handleReset`,
   already existed via `ClientsFiltersBar.onReset` — converged naming, no new UI,
   per Wave 1 EC-5 lesson) / `showAccountantFilter: can.editClients`.
7. **`table` (raw `PaginatedDataTable`).** `data` / `columns` / `onRowClick` /
   `pagination` (`page`/`pageSize`/`total`/`onPageChange`/`onPageSizeChange`) /
   `emptyState`. `emptyState` is an `EmptyStateConfig` carrying
   `isEmpty`/`isFiltered` + the full visual config (`icon`/`variant`/`title`/
   `message`/`action`/`secondaryAction`); the page maps the visual subset into the
   table's `emptyState` prop (no derivation page-side). No `table.selection`
   (Clients has no bulk actions).
8. **`permissions` slot.** `{ can, isAdvisor }` — the feature's `can` object kept
   intact. `can.createClients` gates the create button + empty actions;
   `can.editClients` gates the edit column action + accountant filter + view-only
   Alert; `isAdvisor` threaded into the modal props.

Edge cases preserved: EC-1 (rich empty state — `illustration` variant + create +
import secondaryAction only when truly-empty AND `can.createClients`; header loses
description/actions when `isEmptyState`; stats + filters hidden; filtered-empty /
no-permission fall back to plain "לא נמצאו לקוחות"). EC-2 (`create=1` advisor-only +
`replace`/`preventScrollReset` strip). EC-3 (deleted-client restore interplay —
create modal open only when `!deletedClientDialogOpen`; restore→navigate;
force-create dismiss; dismiss→reopen create). EC-4 (edit drawer loading/error/form;
`onSave` updates then closes; `ModalFormActions` submits via `formId`). EC-5
(permission gating, above). EC-6 (row click → `CLIENT_ROUTES.detail(id)`). EC-7
(cross-feature `ImportExportModal` from `@/features/importExport`).

Sanctioned exceptions kept page-side: the two header action buttons (ייבוא / ייצוא,
לקוח חדש) stay page-composed JSX wired to `modals.openImportExport` /
`modals.openCreate` (same header-action exception as Binders/Users/Charges; header
content branches on the hook's `isEmptyState`); the view-only `Alert` ("צפייה
בלבד…") stays a page-level branch on `permissions.can.editClients`.

Verification: `npm run typecheck` · `npm run lint` (`--max-warnings=0`) ·
`npm run arch:check` (no violations) · `npm run build` (green; pre-existing
large-chunk warning only) · `npm run test` (42 passed / 11 files). No clients unit
tests exist, and the repo still has no `renderHook` / Router+QueryClient hook
harness — focused URL-state / create=1 / restore-flow tests were not cheap to add
(same finding as Waves 1–2) and were not added. Behavior-preserving → diff review
vs `main`; Atlas smoke of `/clients` strongly recommended (rich empty state +
deleted-client restore flow break easily).

---

## Wave 4 — WorkQueuePage

Status: **done.** Behavior identical to `main`; automated verification passed.
`WorkQueuePage` is now pure slot composition (~70 lines, no second hook call,
no `useEffect`, no `useMemo`, no `buildWorkQueueColumns` import, no `renderEmpty`
function) — except the one page-composed "משימה חדשה" header button (sanctioned).
This page is the standard-validator: it already exposed `isFetching` on `main`,
so getting it cleanly into the grouped contract confirms the shape generalizes.

Files changed:
- `features/workQueue/hooks/useWorkQueuePage.ts`
- `features/workQueue/pages/WorkQueuePage.tsx`

Changes:
1. **Grouped contract.** `useWorkQueuePage` now returns
   `status` / `headerProps` / `stats` / `filters` / `table` / `modals`. The old
   flat object (`items`/`summary`/`isLoading`/`isFetching`/`error` + ~10 filter
   fields + handlers + `page`/`total`/`setPage`) is gone.
2. **Second-actions hook composed INTO the page hook.** `useWorkQueueActions()`
   (task-modal / confirm-dialog / mutation state) is now called inside
   `useWorkQueuePage`, mirroring the Binders precedent (`useBindersPageDialogs`
   composed inside `useBindersPage`). `useWorkQueueActions` remains the sole owner
   of task-modal/confirm/mutation **state** — only the call site relocated, so the
   page consumes ONE `useWorkQueuePage()`. This is the "second-actions hook"
   resolution.
3. **Columns into the hook.** `buildWorkQueueColumns` + the `showLinkedTasks` /
   `showWarnings` derivation (from `items`) moved out of the page into the hook
   (columns need `activeActionKey` / `runAction` from the now-composed actions);
   the page no longer imports `buildWorkQueueColumns`.
4. **Error-toast effect into the hook.** The `useEffect` that fires
   `toast.error('טעינת העבודה לטיפול נכשלה', { description })` on error change
   moved into the hook (fires once per `requestError` change). The page owns no
   effect.
5. **`status` group.** `{ isLoading, isFetching, error: requestError,
   loadingMessage: 'טוען משימות...' }`.
6. **`stats` group (EC-5 preserved).** `{ summary, isLoading: isFetching,
   summaryError: error, urgencyFilter, onFilter }` — summary cards intentionally
   use `isFetching` (NOT `isLoading`) and `summaryError = error`; preserved
   exactly. Spread straight into `WorkQueueSummaryCards`.
7. **`filters` group.** All ~10 filter fields (`search` / `urgencyFilter` /
   `typeFilter` / `statusFilter` / `linkedFilter` / `scopeFilter` / `historyMode`
   / `hasFilters` / `hasContentFilters`) + `onFilterChange` / `onMultiFilterChange`
   + `resetFilters` (= `clearFilters`, canonical name) + `onClear` (= `clearFilters`,
   the bar's prop name). Spread straight into `WorkQueueFiltersBar`; the extra
   `hasContentFilters` / `resetFilters` keys are harmless via JSX spread (no excess
   check). `onClear` retained so the spread satisfies the bar without renaming the
   component prop (smallest change).
8. **`table` group (raw `PaginatedDataTable`).** `{ data, columns, isLoading,
   isFetching, pagination {page, pageSize: WORK_QUEUE_PAGE_SIZE, total,
   onPageChange}, label: 'משימות', showPagination: total > 0, stickyHeader: true,
   emptyState }`. Both `isLoading` + `isFetching` passed (EC-7).
9. **`modals` group.** Pre-wired `confirmProps` (open/title/message with the
   `pendingConfirm.action.confirm_title` / `confirm_message` fallbacks, `isLoading:
   actionMutation.isPending`, onConfirm/onCancel) + `taskModalProps` (null when
   closed; mode/source/`task: taskDetail.data`/`isLoading: create||update||detail`/
   onClose/onSubmit) + `openCreateTask`. The page renders `<ConfirmDialog
   {...modals.confirmProps} />` and `{modals.taskModalProps && <TaskModal
   {...modals.taskModalProps} />}`.

### `renderEmpty` → `emptyState` decision + outcome

`renderEmpty` (custom `StateCard`, branched on `hasContentFilters` / `historyMode`)
was converted to an `EmptyStateConfig` computed in the hook (`isEmpty:
items.length === 0`, `isFiltered: hasContentFilters`, `icon: CheckSquare`, branched
`variant` / `title` / `message`): filtered → `default` variant / "אין תוצאות" /
"אין תוצאות שתואמות לסינון"; unfiltered → `illustration` variant / `historyMode ?
'אין היסטוריה' : 'אין עבודה לטיפול'` / `historyMode ? 'אין משימות היסטוריות
להצגה.' : 'אין כרגע עבודה לטיפול. כל הדוחות, התשלומים והמשימות הפעילות מסודרים.'`
The page maps the visual subset (`icon`/`variant`/`title`/`message`) into the
table's `emptyState` prop (no derivation page-side; mirrors Wave 3 Clients).

**Verified rendering equivalence (no fallback to `table.renderEmpty` needed).**
`PaginatedDataTable` → `DataTable` renders the empty `StateCard` from `emptyState`
with the same `icon` / `variant` / `title` / `message` the old `renderEmpty`
returned, and both paths are gated identically: `renderEmpty` was shown only when
`isEmpty && !suppressEmpty` (suppressed during background refetch), and the
`emptyState` path also yields `null` under `suppressEmpty` (= `isFetching &&
isEmpty`) before reaching `DataTable`. During `isLoading` both paths show the
skeleton (`isEmpty` is false). The rendered output is identical, so the cleaner
`emptyState` config was used (no `renderEmpty` passthrough).

Edge cases preserved: EC-1 (350ms `useDebounce` stays in hook), EC-2 (role gating:
no role → `error: 'לא ניתן לזהות תפקיד משתמש'` + query disabled via `enabled:
hasRole`), EC-3 (`historyMode` → empty text + `include_task_history`), EC-4 (error
toast once per error change, now hook-owned), EC-5 (summary cards use `isFetching`
+ `summaryError = error`), EC-6 (columns depend on `showLinkedTasks`/`showWarnings`
+ `activeActionKey`/`runAction`), EC-7 (`label="משימות"`, `showPagination={total >
0}`, `stickyHeader`, both `isLoading` + `isFetching` passed), EC-8 (confirm dialog
fallbacks), EC-9 (TaskModal `isLoading = create || update || detail`), EC-10
(the `parseTaskStatus` `no-restricted-imports` eslint-disable preserved; tasks
barrel not introduced).

Behavior-preserving side-effect:
- Wrapped `items` in `useMemo` (was a bare `data?.items ?? []`) — moving the
  columns memo into the hook surfaced the same unstable-ref concern Binders/Charges
  hit (the `showLinkedTasks`/`showWarnings` + columns memos now depend on `items`).

Sanctioned exception kept page-side: the "משימה חדשה" header button stays
page-composed JSX (label + icon + its `data-work-queue-focus-fallback` attribute),
wired to `modals.openCreateTask` (same header-action exception as
Binders/Users/Charges/Clients).

Verification: `npm run typecheck` · `npm run lint` (`--max-warnings=0`) ·
`npm run arch:check` (no violations) · `npm run build` (green; pre-existing
large-chunk warning only) · `npm run test` (42 passed / 11 files). No workQueue
unit tests exist, and the repo still has no `renderHook` / Router+QueryClient hook
harness — focused URL-state tests were not cheap to add (same finding as Waves
1–3) and were not added. Behavior-preserving → diff review vs `main`; Atlas smoke
of `/work-queue` recommended (verify both empty variants + history-mode text render
identically after the `renderEmpty` → `emptyState` conversion).
