## Scope
This file owns only:
- Backend domain and package inventory for agents.
- Stable backend module classification that helps locate code.

This file must not contain:
- Backend architecture rules, product/domain behavior, endpoint contracts, or implementation plans.

Source of truth: reference

# Backend Module Map

Top-level backend `app/` layout:

```text
app/
|-- main.py
|-- config.py
|-- database.py
|-- lifespan.py
|-- router_registry.py
|-- model_registry.py
|-- <domain>/
|-- common/
|-- core/
|-- infrastructure/
|-- middleware/
|-- actions/
|-- utils/
|-- seed/
`-- tasks/
```

Key top-level files:

| File | Role |
|------|------|
| `app/main.py` | FastAPI app construction, middleware wiring, router registration, public routes, and development storage serving |
| `app/config.py` | pydantic-settings `Settings` class and single `settings` singleton |
| `app/database.py` | SQLAlchemy engine, `SessionLocal`, `Base`, query timing hooks, and `get_db()` dependency |
| `app/lifespan.py` | Startup/shutdown orchestration, including tax calendar bootstrap and daily expiry job |
| `app/router_registry.py` | `register_routers(app)` for mounting domain routers |
| `app/model_registry.py` | Imports ORM model modules for SQLAlchemy mapper setup and Alembic autogenerate |
| `app/users/api/deps.py` | `get_current_user`, `require_role()`, `CurrentUser`, and `DBSession` aliases |
| `app/core/exceptions.py` | Application exception classes and runtime error-envelope builders |
| `app/core/openapi_responses.py` | Shared OpenAPI `responses=` helpers for documented error responses |
| `app/core/exception_handlers.py` | FastAPI handlers for application, validation, and unexpected exceptions |
| `app/core/logging_config.py` | Structured logging formatter, request stats, and request summary logging |
| `app/core/api_types.py` | Shared API types such as `ApiDateTime`, `ApiDecimal`, and `PaginatedResponse` |
| `app/common/repositories/base_repository.py` | Generic CRUD helpers, soft-delete helpers, and pagination support |
| `app/seed/config.py` | `SeedConfig` for development seed behavior |
| `app/seed/orchestrator.py` | Development seed orchestration and schema readiness checks |

`BaseRepository` provides generic `get`, `get_by_ids`, `get_by_id_for_update`, `add`, `build_and_add`, `update`, `soft_delete`, `hard_delete`, `delete`, `apply_pagination`, and `select_base` helpers.

Routed domains have an `api/` router and are mounted under `/api/v1/*`:

`advance_payments`, `annual_reports`, `audit`, `authority_contacts`, `binders`, `businesses`, `charges`, `clients`, `communications`, `dashboard`, `documents/permanent_documents`, `health`, `invoices`, `notes`, `notifications`, `reminders`, `reports`, `search`, `signature_requests`, `tasks`, `tax_calendar`, `timeline`, `users`, `vat`, `work_queue`

Internal-only packages have no HTTP router. Some have a full backend layer stack; others currently expose only the layers needed by callers:

`contacts`, `documents`, `legal_entities`

`alerts` is a logical domain documented in `docs/domains/alerts.md`; it has no `app/alerts/` package. The current attention-board calculation lives in `app/dashboard/services/dashboard_attention_service.py`.

Cross-cutting packages do not follow the domain layer structure:

| Package | Purpose |
|---------|---------|
| `app/common/` | Shared enums, `BaseRepository`, soft-delete helpers |
| `app/core/` | Exceptions, logging, env validation, API types |
| `app/infrastructure/` | Storage provider, notification channels, idempotency |
| `app/middleware/` | RequestIDMiddleware, rate limiting |
| `app/actions/` | UI action metadata registry and cross-domain action orchestration helpers; service modules live under `app/actions/services/` |
| `app/utils/` | General helpers for time, enums, Excel, and ID validation |
| `app/seed/` | Development data seeding orchestrator and builders |

Domain API modules usually use this layout:

```text
app/<domain>/api/
|-- routers.py
`-- <feature>.py
```

`routers.py` assembles sub-routers into the domain router consumed by `register_routers(app)`. Feature router files group related endpoints, such as list/detail operations or lifecycle actions.

Domain sub-routers declare their own prefix, tags, and auth dependencies. Router tags use the domain name for Swagger grouping.

Some read-only or aggregator domains intentionally omit layers they do not need. For example, `dashboard`, `reports`, `search`, `timeline`, and `work_queue` have no ORM models of their own.

`binders` is the reference domain for multiple feature routers:

```text
app/binders/api/
|-- routers.py
|-- binders_list_get.py
|-- binders_receive_return.py
|-- binders_operations.py
|-- binders_audit.py
`-- client_binders_router.py
```
