## Scope

This file owns only:

- Frontend architecture rules that apply across frontend features.
- Frontend ownership boundaries for UI, API, state, schemas, and feature modules.

This file must not contain:

- Product screen specs, domain behavior, backend implementation rules, or API contract definitions.

Source of truth: mandatory

# Frontend Architecture

This document defines frontend ownership and dependency boundaries. Page composition rules live in
`docs/frontend/page-structure.md`; visual and interaction rules live in
`docs/frontend/ui-guidelines.md`.

## Runtime and language

- Frontend code uses React, TypeScript strict mode, Vite, React Router, TanStack Query, Axios,
  react-hook-form, and Zod.
- Do not introduce a second library for routing, server state, forms, validation, HTTP, icons, or
  notifications without an ADR.
- Production code must not use `any`, unchecked response casts, or TypeScript suppression comments
  to bypass a contract problem.
- Use the `@/` alias for imports rooted at `src/`. Relative imports are appropriate within the same
  small module folder.

## Ownership

- Feature-specific UI, hooks, API contracts, endpoint maps, query keys, schemas, constants, and types
  must stay inside `src/features/<feature>/`.
- Shared UI primitives must stay in `src/components/ui/`.
- Shared layout must live in `src/components/layout/`.
- Shared cross-feature widgets must live in `src/components/shared/`.
- App-wide infrastructure belongs in the existing top-level owner: `src/api/`, `src/lib/`,
  `src/router/`, `src/store/`, `src/hooks/`, `src/types/`, or `src/utils/`.
- Do not move feature-specific behavior to a top-level shared folder to avoid a dependency decision.
- Extract code to shared ownership only after there are at least two real cross-feature consumers and
  the extracted API has no domain-specific behavior.

## Dependency boundaries

- Shared UI must not import feature APIs, React Query, auth/session state, or product-specific
  business logic.
- A feature must not deep-import another feature's hooks, API files, schemas, types, constants, or
  utilities. Cross-feature imports must use the owning feature's public `index.ts`.
- Cross-feature component composition is allowed when the consuming screen genuinely owns the
  workflow. Prefer the feature public barrel when the component is part of the feature's public API.
- Do not create circular feature dependencies. Move genuinely shared contracts or infrastructure to
  an appropriate top-level owner instead.
- Architecture exceptions must be narrow, documented beside the import, and removed rather than
  normalized. `arch-check-disable` is not a routine escape hatch.
- If importing through a feature barrel would create or preserve a circular dependency, prefer first
  to move the shared contract to an appropriate owner. If the shared surface is too narrow to move,
  a direct import of that feature's exact contract/constant file is allowed only with a local
  `no-restricted-imports` suppression that explains the barrel cycle. Do not use this exception to
  deep-import hooks, components, or mutable feature behavior.

## UI composition

- Conditional `className` composition must use the shared `cn()` helper from `src/utils/utils.ts`; that helper intentionally wraps `clsx` with `tailwind-merge`, so `clsx` is a direct frontend dependency.
- Reuse primitives from `src/components/ui/` and layout components from `src/components/layout/`
  before adding a new local equivalent.
- Feature components receive data and callbacks through props. Data fetching and mutations belong in
  feature hooks, except for a feature-level widget whose documented responsibility includes loading
  its own resource.
- User-facing copy must be Hebrew where relevant. Internal identifiers, API fields, and code remain
  English.

## Server state and API access

- Server data must use React Query.
- Zustand must be limited to auth/session state unless an explicit ADR says otherwise.
- HTTP calls must go through the shared Axios client.
- API modules contain transport code only: endpoint selection, query serialization, request body, and
  response typing. They must not show toasts, navigate, mutate React state, or contain product
  decisions.
- Query parameters must use the shared serializer when one exists. Do not hand-build query strings.
- Feature request and response types must match the backend contract. The generated OpenAPI file is a
  drift baseline, not a substitute for feature-local contracts and schemas.
- Preserve backend nullability. Do not silently convert `null`, missing fields, empty strings, and
  zero into one another.
- Dates crossing the API boundary are ISO strings. Parse or format them only at the presentation
  boundary; do not change their meaning in the API module.
- Query keys must be feature-local, serializable, and stable.
- Query keys must include every request input that can change the response.
- Search inputs that call the API on user input must debounce requests by default, typically in the 300-500ms range, unless the owning screen has a documented reason to use immediate requests.
- Navigation state such as page, page size, sort, search, and shareable filters must use URL params as the single source of truth.
- Frontend feature code must not keep a second in-memory source of truth for navigation state that already exists in the URL.
- Frontend KPI cards, totals, counters, and sums must come from backend aggregate fields when the underlying dataset is paginated, filtered, or partially loaded.
- Frontend code must not derive product-facing totals from a partial page of server data unless the screen contract explicitly defines the data as complete.
- React Query updates must prefer `setQueryData` for direct cache edits and targeted invalidation for affected entities or exact query keys.
- Do not use broad `invalidateQueries({ queryKey: ...all })` patterns as a default cache strategy when a narrower invalidation or cache update can express the change.
- React Query `staleTime` must match the volatility of the data: effectively static reference data such as enums or config should use `Infinity`, while operational data should use an explicit bounded stale time chosen for that screen.
- Shared stale-time constants should be organized by data volatility and used consistently instead of ad hoc per-hook defaults.
- When multiple features need the same reference-data lookup (e.g. "active users for a dropdown"), extract one shared hook with a semantic, intent-based query key (e.g. `usersQK.activeOptions()`) rather than each feature calling the list endpoint with its own ad hoc params. Only share a hook when the underlying intent is truly identical; do not force-share queries whose filters differ for a real reason (e.g. audit trails needing inactive users).

### React Query freshness policy

Use the semantic constants from `src/lib/queryDefaults.ts`. Select the bucket from the data's
volatility and the user's freshness expectation, not from endpoint shape or as a bulk performance
optimization.

| Bucket | Current value | Intended data |
|---|---:|---|
| `short` | 10 seconds | Operational or live-ish views such as work queues, notifications, pending requests, and operational dashboards |
| `default` | 30 seconds | Normal list and detail data where brief reuse is acceptable |
| `medium` | 1 minute | Slowly changing operational metadata that is revisited frequently |
| `long` | 5 minutes | Low-volatility reference lookups such as active advisor options |
| `static` | `Infinity` | Effectively immutable configuration or enum-style reference data |

- The shared `QueryClient` owns the `default` bucket. Hooks that use normal freshness should inherit
  it and must not add a redundant `staleTime: QUERY_STALE_TIME.default`.
- Hooks must declare an explicit override when their data is more volatile or more stable than the
  global default. Do not apply one bucket to a group of unrelated queries without classifying each
  resource.
- Operational queries must use an explicit bounded bucket. Examples include notifications, work
  queues, pending signature requests, and dashboards used as an operational snapshot.
- Before increasing a query's stale time, audit every frontend mutation that can affect it. Each
  mutation must update the exact cache entry or invalidate the narrowest affected query-key root.
- Frontend invalidation covers only changes performed through that frontend mutation. If data can
  change externally through webhooks, background jobs, another user, or a public workflow, define an
  additional refresh mechanism appropriate to the screen: polling, a push channel, a manual refresh
  action, or a documented remount/reconnect trigger.
- `staleTime` is not polling. It only determines when cached data becomes eligible for refetch.
  Expiration alone does not issue a request.
- `refetchOnWindowFocus` is disabled globally. Do not assume returning to the browser refreshes an
  operational screen; enable an explicit trigger for that query when the product requires it.
- A dashboard must be classified explicitly: use `short` for an operational snapshot and
  `default` for a general overview. Do not choose based only on the cost of its aggregate endpoint.

## Mutations and errors

- Mutation hooks own cache updates, targeted invalidation, and reusable mutation lifecycle behavior.
- A mutation must expose loading and error state to the UI. Prevent duplicate submission while the
  mutation is pending.
- Use the shared error parsing and toast helpers. Do not display raw Axios errors, backend stack
  details, or ad hoc fallback text from individual components.
- Expected validation or conflict errors should be rendered near the relevant form or action when the
  user can correct them. Toasts are for action-level feedback, not a replacement for field errors.
- Destructive or irreversible actions require an explicit confirmation UI.
- Error handling must not swallow a failed request and present stale data as a successful result.

## Authentication and authorization

- Frontend authorization behavior must follow backend permission results.
- Frontend code must handle backend `403` responses consistently with shared UX and must not rely on duplicated role logic as the effective source of truth.
- Access tokens remain in memory. Refresh behavior and cookies are owned by the shared auth client and
  store; feature code must not read or persist tokens.
- Feature code must not add its own Axios auth interceptor or refresh flow.
- Role checks may hide or disable impossible actions for usability, but the backend response remains
  authoritative.

## Forms and validation

- Forms must use react-hook-form and Zod unless existing local code has a narrower established pattern.
- Backend enum-backed fields must use `z.enum([...])`.
- Enum arrays must be defined in constants files, not inline in schemas or components.
- Form value types should be inferred from the Zod schema when practical.
- Keep UI form shape separate from API payload shape when normalization is required. Perform that
  conversion once at the form or mutation boundary.
- Do not send unchanged optional fields, placeholder empty strings, or UI-only values unless the API
  contract explicitly requires them.
- Server validation errors must not be reimplemented as divergent client-only business rules.

## Routing

- Route declarations and guards belong in `src/router/`.
- Feature code must navigate using route constants/builders when the owning feature exports them.
- Pages may read route params and perform route-level guards, but API loading and product logic belong
  in the page hook or feature widget.
- Query-string state must use the shared URL-state helpers and preserve unrelated parameters.
- Use `useSearchParamFilters` for all URL filter/sort/page state. Never hand-roll `new URLSearchParams(searchParams)` + mutate + `setSearchParams` inline — use `setFilter`, `setFilters`, `resetFilters`, or `setPage` instead.
- Read string params via `getParam(key)` (returns `''` when absent). Read the page param via `getPage()`. Do not inline `searchParams.get(key) ?? ''` or `parsePositiveInt(searchParams.get('page'), 1)` at each callsite.

## Verification

- Select verification that matches the changed surface and batch routine checks at a stable
  completion checkpoint. Frontend logic changes commonly require typecheck, lint, and the relevant
  Vitest scope; narrow changes may use a smaller combination when it directly proves correctness.
- Run `npm run arch:check` when imports, ownership boundaries, or shared components change.
- Run `npm run build` for routing, build configuration, dependency, or production-bundle changes.
- Check changed UI in a browser at relevant desktop and mobile widths.
- Use `npm run check` for broad frontend changes. Use `npm run check:strict` for architecture cleanup
  or when cross-feature boundaries are part of the task.
- Do not duplicate UI logic behind wrappers with unclear ownership.
