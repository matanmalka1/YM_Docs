## Scope

This file owns only:

- The mandatory structure, capability, and state contract for client-details tabs
  (the tabs rendered under `/clients/:clientId/<tab>`).

This file must not contain:

- General page structure rules (`docs/frontend/page-structure.md`).
- UI composition / styling rules (`docs/frontend/ui-guidelines.md`).
- Per-feature domain documentation.

Source of truth: mandatory

# Client-Details Tab Contract

Every client-details tab is a scoped view of its feature — not a separate, weaker
implementation. Tabs that predate this contract are drift, not precedent.

## Structure

Every tab renders inside `DetailTabPanel` (`@/components/ui/layout`) using its slots:

| Slot | Rule |
|------|------|
| `title` / `subtitle` | Always set. Feature `<FEATURE>_MESSAGES.clientTab.*` copy. No hand-rolled `h3`/`p` headers. |
| `actions` | Primary action = `Button variant="primary" size="sm"` with an `h-4 w-4` icon. Permission-gated at the same rule as the full page. |
| `summary` | `<Feature>StatsSection` (or equivalent stats component) when the feature has one. Never badge rows or hand-rolled stat markup. |
| `filters` | Shared `FilterPanel` when the tab is filterable. No per-feature filter bars (existing sanctioned exceptions in memory/docs stay). |
| children | Content: table / cards / list. |

Pagination: `PaginatedDataTable` for tables; `PaginationCard` for card/list layouts.

## Capability parity (the core rule)

A tab must expose the same data columns, row actions, drawers, dialogs, and
mutations as the feature's full page, with the client pinned:

- Reuse the full page's `use<Feature>Page` hook, passing a pinned-client option
  (e.g. `useBindersPage({ pinnedClient })`). The hook pins `client_record_id` in
  list params, drops the client filter field, and excludes the client from
  `isFiltered`.
- Reuse the full page's column builders, drawers, and dialog components.
  Never fork a diluted copy (read-only table, inline column defs).
- Client-identity columns (client name, office number, id number) are omitted —
  the page already is the client. Everything else stays.
- If the client-scoped endpoint cannot carry the full response (e.g. no
  `available_actions`), the tab uses the global list endpoint with
  `client_record_id` pinned instead of accepting a weaker response.

## States

| State | Rule |
|-------|------|
| Loading | Section-loading only: header, summary, and filters stay visible; the content area shows the table/card skeleton (`isLoading` on `PaginatedDataTable`, or `TableSkeleton` for non-table content). Never a full-tab `PageLoading`/`PageStateGuard` replace. |
| Error | `Alert variant="error"` (or the table's `error` prop) with retry where a refetch handle exists. Header and filters stay visible. |
| Empty | `emptyState` config (title, message, action) on the table, or `InlineState` for non-table content. The action mirrors the tab's primary action. |

## URL state

Each tab has its own route (`CLIENT_ROUTES.tab`), so filters/page/deep-links use
URL params via `useSearchParamFilters`, same as full pages. Local `useState` for
filters/page is drift (allowed only where the full page itself is not URL-driven).

## Compliance status (tracking only, not part of the contract)

Per-tab compliance tracking lives in `docs/frontend/page-refactor-status.md`.
