## Scope
This file owns only:
- Frontend architecture rules that apply across frontend features.
- Frontend ownership boundaries for UI, API, state, schemas, and feature modules.

This file must not contain:
- Product screen specs, domain behavior, backend implementation rules, or API contract definitions.

Source of truth: mandatory

# Frontend Architecture

- Feature-specific UI, hooks, API contracts, endpoint maps, query keys, schemas, and types must stay inside `src/features/<feature>/`.
- Shared UI primitives must stay in `src/components/ui/`.
- Shared UI must not import feature APIs, React Query feature hooks, or product-specific business logic.
- Shared layout must live in `src/components/layout/`.
- Shared cross-feature widgets must live in `src/components/shared/`.
- Server data must use React Query.
- Zustand must be limited to auth/session state unless an explicit ADR says otherwise.
- HTTP calls must go through the shared Axios client.
- Query keys must be feature-local, serializable, and stable.
- Search inputs that call the API on user input must debounce requests by default, typically in the 300-500ms range, unless the owning screen has a documented reason to use immediate requests.
- Navigation state such as page, page size, sort, search, and shareable filters must use URL params as the single source of truth.
- Frontend feature code must not keep a second in-memory source of truth for navigation state that already exists in the URL.
- Frontend KPI cards, totals, counters, and sums must come from backend aggregate fields when the underlying dataset is paginated, filtered, or partially loaded.
- Frontend code must not derive product-facing totals from a partial page of server data unless the screen contract explicitly defines the data as complete.
- React Query updates must prefer `setQueryData` for direct cache edits and targeted invalidation for affected entities or exact query keys.
- Do not use broad `invalidateQueries({ queryKey: ...all })` patterns as a default cache strategy when a narrower invalidation or cache update can express the change.
- React Query `staleTime` must match the volatility of the data: effectively static reference data such as enums or config should use `Infinity`, while operational data should use an explicit bounded stale time chosen for that screen.
- Shared stale-time constants should be organized by data volatility and used consistently instead of ad hoc per-hook defaults.
- Frontend authorization behavior must follow backend permission results.
- Frontend code must handle backend `403` responses consistently with shared UX and must not rely on duplicated role logic as the effective source of truth.
- Forms must use react-hook-form and Zod unless existing local code has a narrower established pattern.
- Backend enum-backed fields must use `z.enum([...])`.
- Enum arrays must be defined in constants files, not inline in schemas or components.
- Do not duplicate UI logic behind wrappers with unclear ownership.
- User-facing copy must be Hebrew where relevant.
