## Scope
This file owns only:
- Database and SQLAlchemy model conventions.
- Persistence rules that are independent of a specific domain.

This file must not contain:
- Alembic workflow commands, API contract rules, product/domain data rules, or frontend state rules.

Source of truth: mandatory

# Database

- PostgreSQL is the target database for development and production.
- Production must not run on SQLite.
- SQLAlchemy ORM models define application persistence.
- `app/model_registry.py` must import ORM model modules before SQLAlchemy mapper configuration and Alembic autogenerate.
- ORM models inherit from `Base` in `app/database.py`, which subclasses SQLAlchemy `DeclarativeBase`.
- Application queries must use SQLAlchemy 2.0 style ORM/select constructs.
- Application code must not use SQLAlchemy 1.x `Query` API such as `db.query(Model)`.
- SQLAlchemy engines must use `pool_pre_ping=True`.
- SQL echo must remain disabled; SQL visibility must go through configured logging.
- Application schema must not be managed with `Base.metadata.create_all()`.
- `Base.metadata.create_all()` is allowed only for isolated test databases.
- Schema changes must go through Alembic; see `docs/backend/migrations.md`.
- Derived UX state must be computed in services unless a specific ADR requires persistence.
- Soft-delete behavior must be explicit and consistently filtered where applicable.
- Soft-deletable models use `deleted_at` and may use `deleted_by`; repositories must explicitly exclude deleted rows where relevant.
- Enum fields must remain synchronized with frontend Zod enum schemas.
- Python database enums should use `str, Enum` values and `pg_enum()` from `app/utils/enum_utils.py`.
- Timestamp defaults should use `utcnow` from `app/utils/time_utils.py`.
- Indexes and uniqueness constraints belong in model `__table_args__`; partial indexes must account for supported database dialects.
- Database access must stay in repositories.
- Business decisions must not be hidden inside models or repositories.
