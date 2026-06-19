## Scope

This file owns only:

- Backend architecture rules that apply across backend domains.
- Layer responsibility and backend code ownership boundaries.

This file must not contain:

- Product/domain behavior, API contract details, migration commands, or frontend architecture rules.

Source of truth: mandatory

# Backend Architecture

- Backend code must follow Router -> Service -> Repository -> DB.
- Routed backend domains must follow this vertical slice:

```text
app/<domain>/
|-- api/
|-- services/
|-- repositories/
|-- schemas/
`-- models/
```

- Every file inside a domain slice must be prefixed with the domain name; the folder names the layer, the prefix names the domain, and the suffix names the purpose. Examples: `vat/repositories/vat_invoice_repository.py`, `binders/services/binder_lifecycle_service.py`, `clients/api/clients_routes.py`.
  - Layer/purpose suffixes are fixed: `_repository.py`, `_service.py`, `_guards.py`, `_router.py` / `_routers.py`, `_routes.py` / `_routes_<sub>.py`, `_responses.py`, `_serializers.py`, `_messages.py`, `_constants.py`.
  - The primary model and primary schema may use the bare singular domain noun (`charges/models/charge.py`, `clients/schemas/client.py`); additional models/schemas add a purpose suffix (`binders/models/binder_handover.py`, `clients/schemas/client_requests.py`).
  - No bare role-named files (`routes.py`, `responses.py`, `messages.py`, `constants.py`); prefix them (`vat_responses.py`, `vat_messages.py`).
- Domain `schemas/` modules own Pydantic request and response models.
- Paginated list response schemas must subclass `app.core.api_types.PaginatedResponse[T]` rather than re-declaring the `items`/`total`/`page`/`page_size` envelope by hand; add only the extra aggregate field (e.g. `summary`) in the subclass.
- Domain `models/` modules own SQLAlchemy ORM declarations.
- Domain `api/<domain>_routers.py` assembles feature routers into the domain router registered by `app/router_registry.py`.
- Routers must parse requests, apply endpoint-level dependencies, call services, and return responses.
- Route handlers should inject `CurrentUser` and `DBSession`, instantiate the service with the DB session, call one service method, and map the result to the response schema.
- Routers must not contain business branching, orchestration, persistence logic, or DTO assembly beyond request/response wiring.
- Services must own business logic, orchestration, derived state, fine-grained authorization, idempotency checks, and DTO mapping.
- Routers instantiate a fresh service per request with the injected DB session; services are not singletons.
- Services should be split by responsibility in large domains, such as separate read/list services and mutation/lifecycle services.
- Services hold repository instances for the same request DB session.
- Known service-computed derived state includes `days_in_office`, `urgency`, `signals`, and action lists.
- Services map repository output to Pydantic response schemas.
- List services should prefer typed projection dataclasses for multi-table list reads instead of loading full ORM objects for every joined table.
- Services must not run SQL queries directly; they must call repositories.
- Services must not raise FastAPI `HTTPException`; they must raise `AppError` subclasses for domain errors.
- Repositories must own database access only.
- Repositories return ORM model instances or explicit typed projection dataclasses.
- Repositories must not own business decisions, authorization decisions, or cross-domain orchestration.
- Repositories must not import models from other business domains by default.
- Repositories may import another domain model for join clauses only when needed for scoping or typed read projections.
- Repositories must not construct Pydantic response models.
- Repositories must not raise FastAPI `HTTPException`.
- Repositories flush after writes but must not commit transactions.
- `BaseRepository` subclasses must set a class-level `model`.
- Repositories must not re-implement a `BaseRepository` method (e.g. `get_by_id`, `soft_delete`) when the inherited behavior is identical; override only to add real, differing behavior such as a non-`deleted_at` soft-delete column.
- List repository methods should return `(items, count)` when pagination metadata is needed.
- Pagination must use the shared helpers: `BaseRepository.apply_pagination` for SQL `select` statements and `app.core.pagination.paginate_sequence` for already-materialized in-memory lists. Do not hand-write `offset((page - 1) * page_size).limit(...)` or `seq[start:start + page_size]`.
- Count queries must use the same filters as list queries, without pagination.
- Repository method naming should use `get_by_id`, `list_by_*`, `list_*_paginated`, `count_*`, `map_*_by_*`, and `soft_delete` consistently.
- Application query code must use SQLAlchemy ORM/select constructs.
- Application code must not use raw SQL.
- Raw SQL exceptions are Alembic migrations, Alembic environment setup, seed reset scripts, and schema checks.
- Cross-domain writes must be orchestrated in services.
- Cross-domain read aggregation must live only in explicit read-model/query domains or dedicated query services.
- Cross-domain read joins are allowed in repositories only for scoping or typed read projections.
- UI-visible client-scoped list queries must apply the active-client scope where relevant.
- Background jobs must be idempotent.
- Services that coordinate external I/O, such as file uploads or notification sends, must explicitly commit or roll back database changes before the external call.
- Backend business-local current dates must use `app.utils.time_utils.israel_today()`.
- UTC timestamps must use `app.utils.time_utils.utcnow()` or `app.utils.time_utils.utcnow_aware()` according to the column timezone type.
- Plain `date.today()` is allowed only for non-domain tooling or tests that do not assert business-local date behavior.
- Backend messages that surface to users must be Hebrew where relevant.
