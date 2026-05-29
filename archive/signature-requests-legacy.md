## Scope
This file owns only:
- Historical content from `backend/app/signature_requests/README.md` as it existed at audit date 2026-03-22.

This file must not contain:
- Current canonical behavior (that lives in `docs/domains/signature-requests.md`).

Source of truth: historical

# Signature Requests — Legacy README (archived 2026-05-29)

> Original location: `backend/app/signature_requests/README.md`
> Canonical domain doc: `docs/domains/signature-requests.md`

---

The content below is preserved verbatim from the pre-canonical README. Cross-check against the canonical doc before relying on any claim here — some details (e.g. batch expiry scheduler wiring, countersigned PDF storage) were found to be aspirational rather than implemented.

---

## Audit Summary

- Router wiring is valid:
  - Authenticated routes are included under `/api/v1`.
  - Public signer routes are exposed under `/sign/*` without JWT.
- Service/repository flow is consistent and covered by tests.
- Domain naming uses `client_record_id` as the primary anchor; `business_id` is optional context for business-scoped requests and list routes.
- Test suite for this domain passes: `27 passed`.

## Scope

This module provides:
- Signature request creation with expiring signing tokens
- Public signer actions (`view`, `approve`, `decline`)
- Advisor actions (`cancel`, `audit trail`, `pending list`)
- Client-record and business-scoped signature request listing
- Automatic expiry handling for overdue pending requests
- Immutable audit event logging per lifecycle action

## API

Mounted from `app/router_registry.py`:
- Authenticated signature routes: `app.include_router(signature_requests_routers.router, prefix="/api/v1")`
- Public signer routes: `app.include_router(signature_requests_routers.signer_router)`

### Advisor routes (`/api/v1/signature-requests`)

Roles: `ADVISOR`, `SECRETARY`

- `POST /api/v1/signature-requests` — Create request, immediately move to `pending_signature`, generate token and expiry. Returns `signing_token` and `signing_url_hint`.
- `GET /api/v1/signature-requests/pending` — List `pending_signature` requests (paginated).
- `GET /api/v1/signature-requests/{request_id}` — Get request details with embedded audit trail.
- `POST /api/v1/signature-requests/{request_id}/cancel` — Cancel request (allowed from `pending_signature`).

### Client-record listing

Roles: `ADVISOR`, `SECRETARY`

- `GET /api/v1/clients/{client_record_id}/signature-requests`
- Query params: `status` (optional), `page` (default `1`), `page_size` (default `20`, max `100`).

### Public signer routes (`/sign/{token}`)

Auth: no JWT (token-based)

- `GET /sign/{token}` — Records `viewed` audit event.
- `POST /sign/{token}/approve` — Signs request (`pending_signature → signed`).
- `POST /sign/{token}/decline` — Declines request (`pending_signature → declined`).

## Behavior Notes

- Creation generates an unguessable expiring signing token (`secrets.token_urlsafe(32)`); default expiry is 14 days.
- Signing token is cleared after approve/decline/cancel/expire.
- Signer actions require: status `pending_signature`; not expired.
- Runtime expiry: if signer action occurs after expiry, request auto-marked `expired` and `SIGNATURE_REQUEST.EXPIRED` returned.
- Batch expiry: `expire_overdue_requests()` marks overdue pending requests as `expired` and appends audit events.
- Annual report integration: signing an annual-report approval can auto-transition annual report status from `pending_client` to `submitted` (best effort).

## Error Codes

- `SIGNATURE_REQUEST.NOT_FOUND`
- `SIGNATURE_REQUEST.INVALID_TYPE`
- `SIGNATURE_REQUEST.INVALID_STATUS`
- `SIGNATURE_REQUEST.TOKEN_INVALID`
- `SIGNATURE_REQUEST.EXPIRED`
- `BUSINESS.NOT_FOUND`

## Tests

- `tests/signature_requests/api/test_signature_requests.py`
- `tests/signature_requests/api/test_signature_requests_cancel_and_client_list.py`
- `tests/signature_requests/service/test_signature_requests.py`
- `tests/signature_requests/service/test_signer_actions_auto_advance.py`
- `tests/signature_requests/repository/test_signature_request_repository.py`
- `tests/signature_requests/repository/test_signature_request_repository_list_by_client.py`
