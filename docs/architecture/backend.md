## Scope
This file owns only:
- Backend architecture rules that apply across backend domains.
- Layer responsibility and backend code ownership boundaries.

This file must not contain:
- Product/domain behavior, API contract details, migration commands, or frontend architecture rules.

Source of truth: mandatory

# Backend Architecture

- Backend flow is mandatory: Router -> Service -> Repository -> DB.
- Routers parse requests, apply endpoint-level dependencies, call services, and return responses.
- Routers must stay thin and must not contain business branching.
- Services own business logic, orchestration, derived state, fine-grained authorization, idempotency checks, and DTO mapping.
- Repositories own database access only.
- Repositories must not own business decisions or cross-domain orchestration.
- SQLAlchemy ORM/select constructs are the default for application queries. Do not use raw SQL in application code.
- Cross-domain writes belong in services.
- Cross-domain read aggregation is allowed only for explicit read-model/query domains or dedicated query services.
- Background jobs must be idempotent.
- User-facing backend messages that surface to the UI must be Hebrew where relevant.
