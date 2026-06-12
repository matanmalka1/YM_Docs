## Scope
This file owns only:
- Cross-project API contract conventions shared by backend and frontend.
- Public request/response, list, filtering, sorting, and error contract rules.

This file must not contain:
- Backend internal layering, database modeling rules, frontend component rules, or domain-specific endpoint behavior.

Source of truth: mandatory

# API Contracts

- Authenticated application APIs must use `/api/v1/*`.
- Public backend routes may remain unprefixed when they are intentionally outside the authenticated application API surface, such as `/`, `/info`, `/health`, `/ready`, and `/sign/{token}`.
- Public signature routes use `/sign/{token}/*` and do not require authenticated API JWTs.
- Auth endpoints are `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, and `POST /api/v1/auth/logout`.
- Login returns `{ access_token, user }` and sets the refresh-token cookie.
- Refresh returns `{ access_token }`.
- Logout returns `204` and clears the refresh-token cookie.
- Refresh-token cookie attributes and token storage/lifetime policy are defined in `docs/architecture/security.md`, not here.
- Resource URLs must use stable, plural, kebab-case resource names.
- Business action endpoints may use verbs only for real domain commands or transitions.
- `GET` endpoints must not change state.
- Standard list endpoints must support `page`, `page_size`, `sort_by`, and `order`.
- Standard paginated list endpoints should default to `page=1` and `page_size=20`, with page size capped at 100 unless an owning contract says otherwise.
- New list endpoints must not introduce aliases such as `limit`, `offset`, `per_page`, `sort_dir`, or `order_by`.
- Existing list endpoints that still use aliases such as `sort_dir` or `sort_order` should migrate callers to `order` and then remove the old names.
- Filters must be query parameters with stable `snake_case` names.
- List filters should be scalar query parameters unless the endpoint explicitly needs repeated query parameters, such as exclusion lists.
- Search, filter, sort, and pagination parameters must be stable and URL-addressable so a frontend can restore state on refresh, navigation, and shared links without extra in-memory translation.
- Filtering and sorting must happen before pagination.
- Sorting fields must be allowlisted.
- Standard list responses must return `items`, `total`, `page`, and `page_size`.
- List responses may include additional aggregate fields such as counters when the owning endpoint contract defines them.
- Aggregate metadata is allowed only where the list UI actually consumes it; do not force it onto every list or emit empty metadata objects. New list endpoints should name the aggregate block `summary`; existing `stats`/`counters` blocks may remain where already justified by the consuming UI.
- Endpoints that back KPI cards or filtered dashboards must expose the aggregate fields the UI needs, such as totals, counts, or sums, instead of requiring the frontend to infer them from paginated `items`.
- List endpoints must not return the same full detail DTO used by `GET /{id}` when that DTO contains fields the list UI does not render.
- List endpoints must return a dedicated thin row/card DTO, such as `XxxListItem` or `XxxCard`; detail endpoints must return the full detail DTO, such as `XxxResponse` or `XxxDetailResponse`.
- A list DTO may keep a preview field only when the list UI renders that content inline; routing, debug, internal, and other detail-only fields must not appear in list responses.
- The thin-list-DTO rule applies to new endpoints and to endpoints touched during a refactor; it does not require an immediate retroactive migration of all existing domains.
- Every response-bearing endpoint must declare `response_model=`.
- Endpoints with no response body must use HTTP 204.
- Application error responses must use the standard error envelope: `{ "error": { "code": string, "message": string, "details": unknown | null, "request_id"?: string | null } }`.
- Standard application error responses must include `error.code`, `error.message`, and `error.details`.
- Standard application error responses may include `error.request_id` when a request ID is available.
- `error.code` is stable and machine-readable; clients must match on `error.code`, not translated message text.
- Domain error codes use the `DOMAIN.REASON` pattern, such as `BINDER.NOT_FOUND` or `AUTH.INVALID_REFRESH_TOKEN`.
- Documented OpenAPI responses for application errors must reference the shared `ErrorEnvelope` schema; backend routers should use the shared response-doc helpers from `app/core/openapi_responses.py` for 400, 401, 403, 404, 409, and 500 responses.
- Frontend code that needs backend error details must parse the standard envelope through the shared error utility and must not read endpoint-specific response shapes directly.
- SlowAPI rate-limit responses must use the standard error envelope with a rate-limit error code.
- Validation errors must expose enough field/form detail for frontend display without endpoint-specific parsing.
- Validation error details must identify request fields without transport-specific prefixes such as `body.`.
- `AppError` subclasses map to their configured HTTP status, including 404 for not found, 409 for conflict, and 403 for forbidden.
- Permission-denied responses must use a consistent 403 contract so frontend access-control handling can be shared instead of endpoint-specific.
- Request validation errors return 422.
- Database errors and unhandled exceptions return 500 without exposing internals.
- Hebrew `ValueError` messages that are safe for user display return 400.
- Services must raise `AppError` subclasses for domain errors instead of FastAPI `HTTPException`.
- Routers should prefer `AppError` subclasses for domain errors; transport-bound dependencies such as `get_current_user` may raise `HTTPException`.
- API contracts must use `snake_case` fields to match the Python backend.
- Response schemas should use `ApiDateTime` and `ApiDecimal` from `app/core/api_types.py` where practical.
- `ApiDateTime` serializes as UTC timestamps such as `2026-01-02T03:04:05Z`.
- `ApiDecimal` serializes decimal values as strings with two decimal places to avoid float precision loss.
- Frontend request/response types must match backend contracts.
- Frontend code must not silently remap API contracts to different field names.
