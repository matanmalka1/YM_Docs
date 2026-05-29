## Scope
This file owns only:
- The accepted backend layering decision.

This file must not contain:
- Full backend architecture documentation or domain-specific service behavior.

Source of truth: historical

# ADR 0002: Router-Service-Repository

Decision: Backend code follows Router -> Service -> Repository -> DB.

Consequences:

- Routers stay thin.
- Services own business logic and orchestration.
- Repositories own database access only.
- Cross-domain orchestration belongs in services.
- Business branching does not belong in routers.
