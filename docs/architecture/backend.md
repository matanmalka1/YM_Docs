## Scope
This file owns only:
- Backend architecture rules that apply across backend domains.
- Layer responsibility and backend code ownership boundaries.

This file must not contain:
- Product/domain behavior, API contract details, migration commands, or frontend architecture rules.

Source of truth: mandatory

# Backend Architecture

- Backend code must follow Router -> Service -> Repository -> DB.
- Routers must parse requests, apply endpoint-level dependencies, call services, and return responses.
- Routers must not contain business branching, orchestration, persistence logic, or DTO assembly beyond request/response wiring.
- Services must own business logic, orchestration, derived state, fine-grained authorization, idempotency checks, and DTO mapping.
- Repositories must own database access only.
- Repositories must not own business decisions, authorization decisions, or cross-domain orchestration.
- Application query code must use SQLAlchemy ORM/select constructs.
- Application code must not use raw SQL.
- Cross-domain writes must be orchestrated in services.
- Cross-domain read aggregation must live only in explicit read-model/query domains or dedicated query services.
- Background jobs must be idempotent.
- Backend messages that surface to users must be Hebrew where relevant.
