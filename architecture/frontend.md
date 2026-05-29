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
- Forms must use react-hook-form and Zod unless existing local code has a narrower established pattern.
- Backend enum-backed fields must use `z.enum([...])`.
- Enum arrays must be defined in constants files, not inline in schemas or components.
- Do not duplicate UI logic behind wrappers with unclear ownership.
- User-facing copy must be Hebrew where relevant.
