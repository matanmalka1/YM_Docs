## Scope
This file owns only:
- Detailed public API contract examples and conventions.
- Worked request/response, URL, DTO, and action-naming illustrations subordinate to the canonical API contract rules.

This file must not contain:
- API rules that override `docs/architecture/api-contracts.md`.
- Legacy compatibility policy that conflicts with `docs/adr/0001-no-legacy-compatibility.md`.
- Frontend implementation rules.

Source of truth: reference

Canonical rules:
- `docs/architecture/api-contracts.md`
- `docs/workflow/verification.md`

# API Contract Standard

This file is non-normative. Canonical API rules live in `docs/architecture/api-contracts.md`. If there is conflict, `api-contracts.md` wins.

This document gives worked examples for the public API contract. Internal implementation may differ by domain, but externally visible request and response contracts must be consistent.

## Applicability

This standard applies to endpoints that expose application resources through `/api/v1/*`.

It is mandatory for list endpoints that power tables, queues, feeds, cards with paging, and selectable resource lists.
It does not need to apply to:

- `GET /{id}` single-resource reads
- dashboard summaries and statistics
- exports and downloads
- auth flows
- narrow autocomplete/search endpoints with their own documented contract
- write-only action endpoints that do not return lists

## URL Naming

URL naming rules (plural/kebab-case resources, verbs only for real domain commands) live in `docs/architecture/api-contracts.md`. Examples:

Good:

```txt
/api/v1/clients
/api/v1/clients/{client_id}
/api/v1/binders/{binder_id}/mark-ready-for-handover
/api/v1/notifications/preview
/api/v1/notifications/send
```

Do not use RPC-style or mixed-case paths:

```txt
/getClients
/client/list
/binderReady
/send_notification
```

## HTTP Methods

Method semantics and the rule that `GET` must not change state are defined in `docs/architecture/api-contracts.md`. Quick reference:

| Method | Use |
|--------|-----|
| `GET` | read/list only |
| `POST` | create or business action |
| `PATCH` | partial update |
| `PUT` | full replacement, rarely needed |
| `DELETE` | delete or soft delete |

Bad:

```txt
GET /binders/{binder_id}/complete
```

Good:

```txt
POST /binders/{binder_id}/complete
```

## List Request Contract

The canonical list-parameter rules, defaults, bounds, and prohibited aliases (`limit`/`offset`, `per_page`, `sort_dir`, `order_by`, etc.) are defined in `docs/architecture/api-contracts.md`. A standard list request using `page`, `page_size`, `sort_by`, and `order` looks like:

```txt
GET /api/v1/annual-reports?page=1&page_size=25&sort_by=created_at&order=desc
```

## Filtering

Filter rules (query params, stable snake_case names, filter before pagination) live in `docs/architecture/api-contracts.md`. Preferred names in this system:

```txt
client_record_id
business_id
status
date_from
date_to
query
```

Do not mix aliases for the same concept:

```txt
clientId
client_id
client_record
customer_id
```

In this system, when the identifier points to `ClientRecord`, the API field is `client_record_id`.

## Sorting

Sorting fields must be allowlisted (see `docs/architecture/api-contracts.md`). Map `sort_by` through an allowlist in the service/repository layer rather than passing free-form column names to SQL. Example:

```txt
sort_by=created_at&order=desc
```

## List Response Contract

The required base fields (`items`, `total`, `page`, `page_size`) and the rule that extra aggregate fields may be added but base fields not renamed live in `docs/architecture/api-contracts.md`. Example:

```json
{
  "items": [],
  "total": 120,
  "page": 1,
  "page_size": 25
}
```

Here `total` is the number of records after filters and before pagination. Example with an added aggregate:

```json
{
  "items": [],
  "total": 120,
  "page": 1,
  "page_size": 25,
  "counters": {}
}
```

## Single Response Contract

Single-resource reads return the DTO directly:

```json
{
  "id": 1,
  "name": "..."
}
```

Do not wrap single resources in inconsistent shapes such as `{ "data": ... }` unless the whole API is migrated to a global envelope.

## Action Response Contract

Business actions should usually return the updated DTO when the frontend needs fresh state.

Example:

```txt
POST /api/v1/binders/{binder_id}/mark-ready-for-handover
```

Response:

```json
{
  "id": 12,
  "status": "ready_for_handover"
}
```

If an action does not naturally return a resource, use a consistent result DTO:

```json
{
  "status": "success",
  "message": "הפעולה בוצעה",
  "data": {}
}
```

## Error Contract

Errors use the project envelope. The envelope shape, error codes, and status mapping are defined in `docs/architecture/api-contracts.md`. Example:

```json
{
  "error": {
    "code": "BINDER.NOT_FOUND",
    "message": "קלסר לא נמצא",
    "details": null,
    "request_id": "..."
  }
}
```

## Status Codes (quick reference)

Canonical status rules live in `docs/architecture/api-contracts.md`. Common cases:

| Status | Meaning |
|--------|---------|
| `401` | user is not authenticated |
| `403` | user is authenticated but not allowed |
| `404` | resource does not exist or is intentionally hidden |
| `409` | business conflict |
| `422` | request validation failed |
| `500` | unexpected server error only |

## DTO Boundaries

The `response_model=` requirement is defined in `docs/architecture/api-contracts.md`. In practice, routers return Pydantic response schemas, not raw ORM objects.

Good:

```python
@router.get("/{client_id}", response_model=ClientResponse)
def get_client(...):
    return service.get_client(client_id)
```

Bad:

```python
return db_client
```

Services may return DTOs or ORM instances depending on existing domain patterns, but the router response model is the public contract.

## Business Action Naming

Use explicit verbs for business transitions:

```txt
POST /binders/{binder_id}/mark-ready-for-handover
POST /binders/{binder_id}/return-to-client
POST /tasks/{task_id}/complete
POST /tasks/{task_id}/cancel
POST /notifications/preview
POST /notifications/send
```

Avoid vague action names:

```txt
POST /binders/{binder_id}/ready
POST /binders/{binder_id}/status
POST /tasks/{task_id}/done
```

## Frontend Expectations

For primary table pages, the frontend should:

- send all filters to the backend
- send `page`, `page_size`, `sort_by`, and `order`
- render pagination from `total`
- avoid in-memory pagination for primary resource screens
- use in-memory slicing only for already-small embedded lists where the API explicitly returns the full set

## Migration

Existing endpoints may deviate from this standard. When touching a list endpoint, migrate it toward this contract by updating its callers and removing the old names rather than adding another variation. The no-legacy-compatibility policy is defined in `docs/adr/0001-no-legacy-compatibility.md`.
