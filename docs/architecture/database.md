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
- Use SQLAlchemy 2.0 style ORM/select constructs.
- Do not manage application schema with `Base.metadata.create_all()`.
- Derived UX state such as urgency, signals, and action lists should be computed in services unless a specific ADR says it must be persisted.
- Soft-delete behavior must be explicit and consistently filtered where applicable.
- Enum fields must remain synchronized with frontend Zod enum schemas.
- Database access belongs in repositories.
- Business decisions must not be hidden inside models or repositories.
