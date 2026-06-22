## Scope

This file owns only:

- A lookup index mapping a recurring frontend need to its one canonical helper and import path.

This file must not contain:

- The rules that govern *when* to apply each helper. Those live in `docs/frontend/architecture.md`,
  `docs/frontend/page-structure.md`, and `docs/frontend/ui-guidelines.md` and remain the source of
  truth. This file only points at the existing helper so an agent does not rediscover or hand-roll it.

Source of truth: reference (the linked architecture/page-structure/ui-guidelines rules win on conflict)

# Frontend Canonical Helpers

Before writing display, URL-state, query, error, or UI-shell code, find the need below and use the
listed helper. Do not hand-roll an equivalent, and do not add a feature-local copy. A pattern used by
two or more features must be a shared helper, not duplicated.

## Display formatting — `src/utils/utils.ts`

| Need | Helper | Never instead |
|------|--------|---------------|
| Date `dd/MM/yyyy` | `formatDate` | `toLocaleDateString`, inline `format()` |
| Date+time `dd/MM/yyyy HH:mm` | `formatDateTime` | `Intl.DateTimeFormat` |
| Month/year `MM.yyyy` | `formatMonthYear` | inline `date-fns` |
| Audit `d MMM yyyy HH:mm` | `formatAuditTimestamp` | — |
| Hebrew long date (takes a `Date`) | `formatHebrewDate` | — |
| Shekel currency (`5,000 ₪`) | `formatCurrencyILS` / `formatShekelAmount` | `₪${n.toLocaleString()}`, `₪${n.toFixed(2)}` |
| Percent | `formatPercent` (`isRatio` for `0–1`) | `(n*100).toFixed(d)%` |
| Integer count (`he-IL` grouping) | `formatCount` | `n.toLocaleString()` |
| Error → user string | `getErrorMessage(error, fallback)` | raw Axios error text, ad hoc fallback |

A new display pattern none of these cover → add a named formatter routed through `formatSafeDate`
(dates) or `toNumberOrNull` (numbers) in `utils.ts`, never an inline call site. Exemptions
(calc/form-input values, chart-K ticks, bare `${x}%` echoes) are listed in `architecture.md`.

## URL / navigation state — `src/hooks/useSearchParamFilters.ts`

| Need | Helper |
|------|--------|
| Read string param | `getParam(key)` (returns `''` when absent) |
| Read page param | `getPage()` |
| Write one filter | `setFilter` |
| Write several | `setFilters` |
| Reset all | `resetFilters` |
| Set page | `setPage` |

Never `new URLSearchParams(searchParams)` + mutate + `setSearchParams` inline. Enum-backed params are
untrusted strings — parse with a feature guard, never `as SomeEnum`.

## Pagination — `src/utils/paginationUtils.ts`

| Need | Helper | Never instead |
|------|--------|---------------|
| Total pages from `total` + `pageSize` | `getTotalPages(total, pageSize)` | `Math.max(1, Math.ceil(total / pageSize))` inline |

## React Query freshness — `src/lib/queryDefaults.ts`

Use the `QUERY_STALE_TIME` buckets (`short`/`default`/`medium`/`long`/`static`). The `QueryClient`
owns `default`; do not restate it on a hook. Pick the bucket from data volatility, not endpoint shape.
Full policy + bucket table in `architecture.md`.

## Status badges / labels — `src/utils/labels.ts`

| Need | Helper |
|------|--------|
| Direct badge variant lookup | `makeVariantGetter` |
| Badge component (accepts feature `variantMap`) | `StatusBadge` (`src/components/ui/primitives/`) |

Feature-owned status→variant maps stay with the owning feature's constants.

## UI shells & states — `src/components/ui/`

| Need | Component | Path |
|------|-----------|------|
| Reusable card/container chrome | `Card` | `primitives/` |
| Guarded page load/error wrap | `PageStateGuard` | `layout/` |
| Section/page loading (wraps `TableSkeleton`) | `PageLoading` | `layout/` |
| Paginated list table | `PaginatedDataTable` | `table/` |
| Table loading skeleton | `TableSkeleton` | `table/` |
| Row actions | `RowActions` | `table/` |
| Bulk selection bar | `BulkSelectionToolbar` | `table/` |
| Destructive confirm | `ConfirmDialog` (keep `closeOnBackdrop=false`) | `overlays/` |
| Detail/edit drawer | `DetailDrawer` | `overlays/` |
| Focused create/confirm modal | `Modal` | `overlays/` |
| Empty / error / no-results card | `StateCard` | `feedback/` |
| Inline empty/error/no-results state inside an existing panel | `InlineState` | `feedback/` |
| Unsaved-changes guard | `UnsavedChangesGuard` | `feedback/` |

Use `Card`'s `disablePadding` and `bodyClassName` props for table/full-bleed content instead of
wrapping a card in another card-like container or trying to cancel its body padding from the root.

## Filters and form controls

| Need | Component | Path |
|------|-----------|------|
| Standard filter/search/date/client toolbar | `FilterPanel` with `FilterFieldDef` | `src/components/ui/filters/FilterPanel.tsx` |
| Select dropdown | `Select` | `src/components/ui/inputs/` |
| Labeled checkbox | `Checkbox` | `src/components/ui/primitives/Checkbox.tsx` |

Do not hand-roll a filter bar, plain `<select>`, or labeled raw checkbox when these components cover
the interaction. Native radio inputs and hidden file inputs are allowed when the native control is
intentional and accessible.

## Loading indicators (canonical shapes)

| Need | Use | Path |
|------|-----|------|
| Bar/block placeholder | `SkeletonBlock` | `primitives/` |
| Table skeleton | `TableSkeleton` | `table/` |
| Ring spinner (size + label) | `Spinner` | `primitives/` |
| Inline icon spinner | `Loader2` | `lucide-react` |

Never hand-roll `animate-pulse` / `animate-spin` ring `div`s. `RefreshCw` is a refresh action icon,
not a loader.

## Mutations / toasts

| Need | Helper | Path |
|------|--------|------|
| Mutation + success/error toast (standard case) | `useMutationWithToast` | `src/hooks/` |
| Imperative toast | `toast.{success,error,...}` | `src/utils/toast.ts` |

`useMutationWithToast` does not fit inline-error, optimistic, conditional-invalidate, modal-sequence,
or result-inspection mutations — keep those as raw `useMutation` with the shared error/toast helpers.

## Class composition

`cn()` from `src/utils/utils.ts` (wraps `clsx` + `tailwind-merge`, last-wins). Never concatenate
class strings by hand for conditional classes.
