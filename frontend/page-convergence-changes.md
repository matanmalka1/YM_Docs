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
