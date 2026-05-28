## Scope
This file owns only:
- Frontend architecture rules that apply across frontend features.
- Frontend ownership boundaries for UI, API, state, schemas, and feature modules.

This file must not contain:
- Product screen specs, domain behavior, backend implementation rules, or API contract definitions.

Source of truth: mandatory

# Frontend Architecture

- Keep feature-specific UI, hooks, API contracts, endpoint maps, query keys, schemas, and types inside `src/features/<feature>/`.
- Keep shared UI primitives in `src/components/ui/`.
- Shared UI must not import feature APIs, React Query feature hooks, or product-specific business logic.
- Shared layout belongs in `src/components/layout/`.
- Shared cross-feature widgets belong in `src/components/shared/`.
- Server data belongs in React Query.
- Zustand is for auth/session state only unless an explicit architecture decision says otherwise.
- All HTTP should go through the shared Axios client.
- Use feature-local query keys and keep them serializable and stable.
- Use react-hook-form and Zod for forms.
- Backend enum-backed fields must use `z.enum([...])`; define enum arrays in constants files, not inline in schemas or components.
- Avoid duplicated UI logic and unclear wrappers.
- User-facing copy should be Hebrew where relevant.
