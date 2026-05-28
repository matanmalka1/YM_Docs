## Scope
This file owns only:
- Cross-project API contract conventions shared by backend and frontend.
- Public request/response, list, filtering, sorting, and error contract rules.

This file must not contain:
- Backend internal layering, database modeling rules, frontend component rules, or domain-specific endpoint behavior.

Source of truth: mandatory

# API Contracts

- Authenticated application APIs use `/api/v1/*`.
- Resource URLs use stable, plural, kebab-case resource names.
- Business action endpoints may use verbs only for real domain commands or transitions.
- Never use `GET` for state-changing operations.
- Standard list endpoints must support `page`, `page_size`, `sort_by`, and `order`.
- Do not introduce list aliases such as `limit`, `offset`, `per_page`, `sort_dir`, or `order_by`.
- Filters are query parameters and use stable `snake_case` names.
- Filtering and sorting must happen before pagination, preferably in backend SQL/ORM queries.
- Sorting fields must be allowlisted.
- Standard list responses return `items`, `total`, `page`, and `page_size`.
- API contracts use `snake_case` fields to match the Python backend.
- Frontend request/response types must match backend contracts instead of remapping silently.
