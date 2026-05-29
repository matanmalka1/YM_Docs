## Scope
This file owns only:
- Canonical current-state documentation for the health domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Health

The health domain exposes unauthenticated liveness and readiness probes for the backend process. It does not own business data; its single responsibility is to translate database connectivity into a stable two-field payload and the matching HTTP status for monitoring systems.
Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/health` | Liveness/health probe that reports process + database connectivity. |
| `GET` | `/ready` | Readiness probe for platform checks using the same connectivity test. |

The router defines both handlers without a module prefix at `backend/app/health/api/health.py:21-54`, and `register_routers()` mounts the health router without `/api/v1` at `backend/app/router_registry.py:30-32`. Both effective paths exist in `backend/openapi.json:44-107`.

## Model & fields

This domain has no `backend/app/health/models/` package and no health-owned persisted SQLAlchemy models; `backend/app/health` contains only `api/`, `repositories/`, `schemas.py`, `services/`, and the local README (`rg --files backend/app/health`).

The domain-owned response schema is:

- `HealthCheckResponse` (`backend/app/health/schemas.py:6-8`): `status` with exact literals `healthy | unhealthy`, and `database` with exact literals `connected | disconnected`.

## Enums / statuses

Health does not define Python enum classes, but it does enforce these exact literal status values in schema + service code:

- `status`: `healthy`, `unhealthy` (`backend/app/health/schemas.py:6-8`, `backend/app/health/services/health_service.py:12-20`).
- `database`: `connected`, `disconnected` (`backend/app/health/schemas.py:6-8`, `backend/app/health/services/health_service.py:12-20`).

## Domain rules & invariants

- Both endpoints are public: neither route includes `require_role(...)` or any auth dependency; they depend only on `get_db` (`backend/app/health/api/health.py:10-16,21-54`).
- `GET /health` and `GET /ready` both call the same `HealthService.check()` method, so they have identical semantics and payload shape (`backend/app/health/api/health.py:37-40,49-54`).
- `HealthRepository.can_connect()` performs a lightweight `select(1)` through the SQLAlchemy session and must not use raw SQL (`backend/app/health/repositories/health_repository.py:11-20`).
- Repository-level connectivity failures are normalized to `False` inside `can_connect()`; the service also defensively catches unexpected exceptions from the repository and still reports an unhealthy result (`backend/app/health/repositories/health_repository.py:17-21`, `backend/app/health/services/health_service.py:12-20`).
- The HTTP contract is binary: if `result["status"] != "healthy"`, the route returns `503 Service Unavailable`; otherwise it returns `200 OK` with the same two-field payload (`backend/app/health/api/health.py:37-40,51-54`; `backend/openapi.json:52-73,84-105`).

## Error codes

No `HEALTH.*` domain error codes are raised inside `backend/app/health`; `rg -n 'HEALTH\\.|health\\.' backend/app/health` returns only imports, symbols, and route/schema names. Health returns explicit payloads instead of the app error envelope. Error-code registry: `docs/architecture/error-codes.md`.

## Decisions (preserved)

- Health remains a thin infrastructure domain with no persisted models of its own; the current module contains only router/service/repository/schema pieces and no `models/` package (`rg --files backend/app/health`, `backend/app/health/api/health.py:21-54`, `backend/app/health/services/health_service.py:6-20`, `backend/app/health/repositories/health_repository.py:5-20`, `backend/app/health/schemas.py:6-8`).
- Health/readiness responses stay intentionally small and operationally stable: only `status` and `database` are returned, with `200` for healthy and `503` for unhealthy (`backend/app/health/api/health.py:21-54`, `backend/app/health/schemas.py:6-8`).

## Future / planned

No health-specific future/planned behavior could be verified from `backend/docs/domain_decisions_v3.md`, `backend/docs/**`, or the current module code. Nothing is documented here as current unless it is implemented in code.

## Historical notes

Historical module notes were archived to `docs/archive/health-legacy.md`.
