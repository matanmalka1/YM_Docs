## Scope
This file owns only:
- Cross-project API contract conventions shared by backend and frontend.
- Public request/response, list, filtering, sorting, and error contract rules.

This file must not contain:
- Backend internal layering, database modeling rules, frontend component rules, or domain-specific endpoint behavior.

Source of truth: mandatory

# API Contracts

- Authenticated application APIs must use `/api/v1/*`.
- Resource URLs must use stable, plural, kebab-case resource names.
- Business action endpoints may use verbs only for real domain commands or transitions.
- `GET` endpoints must not change state.
- Standard list endpoints must support `page`, `page_size`, `sort_by`, and `order`.
- New list endpoints must not introduce aliases such as `limit`, `offset`, `per_page`, `sort_dir`, or `order_by`.
- Filters must be query parameters with stable `snake_case` names.
- Filtering and sorting must happen before pagination.
- Sorting fields must be allowlisted.
- Standard list responses must return `items`, `total`, `page`, and `page_size`.
- Application error responses must use the standard error envelope.
- Standard application error responses must include `error.code`, `error.message`, and optional `error.details`.
- Validation errors must expose enough field/form detail for frontend display without endpoint-specific parsing.
- API contracts must use `snake_case` fields to match the Python backend.
- Frontend request/response types must match backend contracts.
- Frontend code must not silently remap API contracts to different field names.
