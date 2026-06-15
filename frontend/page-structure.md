## Scope

This file owns only:
- The standard structure for frontend feature pages.
- Rules that apply to every page component, hook, component, and API file inside a feature.

This file must not contain:
- Product/domain behavior rules.
- Backend architecture rules.
- API contract definitions.

Source of truth: mandatory

---

# Frontend Page Structure Standard

Read this file before refactoring any page.

## Folder structure

Every feature follows this layout:

```
features/<domain>/
  api/
    <domain>Api.ts       — HTTP calls only, no business logic
    contracts.ts         — request/response types, Zod schemas
  pages/
    <Domain>Page.tsx     — thin orchestrator, no API calls, no logic
  hooks/
    use<Domain>Page.ts   — primary page hook: query, filters, mutations, derived state
    use<Domain>Filters.ts  — filter shape + URL sync (when filter logic is complex)
    use<Domain>Actions.ts  — mutations + confirmation state
  components/
    <Domain>PageHeader.tsx
    <Domain>FiltersBar.tsx
    <Domain>Table.tsx
    <Domain>Drawer.tsx
    <Domain>EmptyState.tsx
  utils/
    <domain>Formatters.ts  — pure formatting/display helpers
```

Not every file is required. Create only what the domain actually needs.

---

## Page component rules

- Page = orchestration only. Layout + slot-filling. Nothing else.
- No direct `useQuery` or `useMutation` calls inside Page.
- No API calls inside Page.
- Page consumes one page-level hook: `useXPage()`.
- Exception: a second hook is allowed when it has a clearly separate concern (e.g. `useXActions()` for confirmations).
- Usually under ~80 lines. Exception allowed if the page only composes sections and contains no logic.
- If the page already matches this standard, do not refactor it. Document it as compliant.

```tsx
// Good
export function ClientsPage() {
  const page = useClientsPage()
  return (
    <PageShell>
      <ClientsPageHeader total={page.total} onCreate={page.openCreate} />
      <ClientsFiltersBar filters={page.filters} onChange={page.setFilters} />
      <ClientsTable clients={page.clients} isLoading={page.isLoading} onRowClick={page.openClient} />
      <ClientsDrawer client={page.selected} open={page.drawerOpen} onClose={page.closeDrawer} />
    </PageShell>
  )
}
```

---

## Hook rules

- `use<Domain>Page` owns: page orchestration, query state, filter state, derived state, modal/drawer state. It may call `use<Domain>Actions`, but should not become a mutation dump.
- `use<Domain>Filters` owns: filter shape, URL sync, reset logic. Use when filter logic exceeds ~30 lines.
- `use<Domain>Actions` owns: mutations, confirmation dialogs, optimistic updates.
- Hooks must not render JSX.
- Hooks must not import from other feature hooks unless there is an explicit cross-domain dependency.

---

## Component rules

- Components receive data via props. They do not fetch their own data.
- Components do not call `useQuery`, `useMutation`, or feature API hooks directly.
- Exception: a documented feature-level widget may load the resource it owns. Keep its presentational
  children fetch-free.
- Shared UI primitives live in `src/components/ui/`.
- Shared cross-feature widgets live in `src/components/shared/`.
- Do not move domain-specific UI into shared.
- Extract to shared only after 2–3 real usages across different domains.
- Shared must not import feature APIs, React Query feature hooks, or business logic.
- Feature widgets may use their own feature-level hooks, but presentational components must not fetch.

---

## API layer rules

- `<domain>Api.ts` contains HTTP calls only. No business logic, no state.
- `contracts.ts` contains request/response types and Zod schemas.
- Endpoint constants and query-key factories remain feature-local.
- API methods must return typed data and preserve backend nullability.
- Use the shared Axios client and shared query-param serializer.
- API modules must not show toasts, navigate, or invalidate React Query caches.
- Reusable enum option arrays should live in constants files, not inline in multiple components.
- HTTP calls must go through the shared Axios client.
- Query keys must be feature-local, serializable, and stable.

---

## Import rules

- Use `@/` for imports outside the current module folder.
- A feature may import its own internals directly.
- Import another feature's non-component API only from that feature's public `index.ts`.
- Do not deep-import another feature's hooks, API, schemas, types, constants, or utilities.
- Pages may compose exported cross-feature components. Reusable cross-feature logic still belongs
  behind the owning feature's public barrel.
- Keep every feature `index.ts` intentional: export only the surface other features are allowed to
  depend on.

---

## URL state rules

- search / filter / sort / page / page_size that should survive refresh → URL params.
- Modal open state → local state (unless product requires deep-linkable modals).
- Selected drawer/row id → URL param only if the product requires a shareable link.
- Never duplicate a URL param into local state as a shadow copy.
- `useSearchParamFilters` is the standard hook for URL filter sync. Use its `setFilter` / `setFilters` / `resetFilters` / `setPage` helpers for all writes, and `getParam(key)` / `getPage()` for reads. Never hand-roll `new URLSearchParams(searchParams)` mutations inline.
- URL params and `<select>` values are external strings. Enum-backed filters must parse them with a
  feature-owned guard/parser before assigning typed state or building API params. Invalid enum values
  should normalize to the presentation sentinel (`''`) or omission (`undefined`/`null`), not be cast
  with `as SomeEnum`.

---

## List page standard

Every List page must handle all of these consistently:

| Concern | Implementation |
|---------|---------------|
| `search` | Debounced 300–500ms, URL param |
| `filters` | URL params, reset-all supported |
| `sort_by` / `order` | URL params (sort direction param is `order` with `asc`/`desc`, per `docs/architecture/api-contracts.md`) |
| `page` / `page_size` | URL params (or cursor if backend uses cursor) |
| Loading state | Skeleton or spinner |
| Empty state | Distinct `<XEmptyState />` component |
| Error state | Error message + retry action |

---

## Page type classification

Classify every page before refactoring:

| Type | Characteristics |
|------|----------------|
| **List** | Table, filters, pagination, bulk actions |
| **Detail** | Header, tabs/sections, drawers, related entities |
| **Dashboard** | Widgets, cross-feature composition |
| **Public** | No auth shell, different loading/error behavior |
| **Settings** | Forms, save/reset, dirty state, optimistic updates |
| **Wizard/Form** | Multi-step state, validation, draft state |

Type determines which hooks and components are expected.

---

## Naming rules

- Names must be explicit and domain-specific.
- No generic names: `BaseCrudPage`, `GenericTable`, `usePageHook`.
- Do not rename files unless the rename is required by this standard.
- Prefer moving logic over cosmetic reshuffling.

---

## What not to do

- Do not call API hooks directly from Page.
- Do not keep a second in-memory copy of URL filter state.
- Do not derive product-facing totals from a partial page of server data.
- Prefer targeted cache updates/invalidation. Avoid broad `invalidateQueries` unless the mutation affects multiple unknown views.
- Do not create `BaseCrudPage` or any generic mega-abstraction.
- Do not extract to shared before 2–3 real cross-domain usages.
- Do not refactor a page that already matches this standard.
