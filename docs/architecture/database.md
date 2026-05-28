## Scope
This file owns only:
- Database and SQLAlchemy model conventions.
- Persistence rules that are independent of a specific domain.

This file must not contain:
- Alembic workflow commands, API contract rules, product/domain data rules, or frontend state rules.

Source of truth: mandatory

# Database

- PostgreSQL is the target database for development and production.
- SQLAlchemy ORM models define application persistence.
- Application queries must use SQLAlchemy 2.0 style ORM/select constructs.
- Application schema must not be managed with `Base.metadata.create_all()`.
- Schema changes must go through Alembic; see `docs/architecture/migrations.md`.
- Derived UX state must be computed in services unless a specific ADR requires persistence.
- Soft-delete behavior must be explicit and consistently filtered where applicable.
- Enum fields must remain synchronized with frontend Zod enum schemas.
- Database access must stay in repositories.
- Business decisions must not be hidden inside models or repositories.
