## Scope
This file owns only:
- Database and SQLAlchemy model conventions.
- Persistence rules that are independent of a specific domain.

This file must not contain:
- Alembic workflow commands, API contract rules, product/domain data rules, or frontend state rules.

Source of truth: mandatory

# Database

- PostgreSQL is the only supported database for development, tests, and production.
- SQLAlchemy ORM models define application persistence.
- `app/model_registry.py` must import ORM model modules before SQLAlchemy mapper configuration and Alembic autogenerate.
- ORM models inherit from `Base` in `app/database.py`, which subclasses SQLAlchemy `DeclarativeBase`.
- Application queries must use SQLAlchemy 2.0 style ORM/select constructs.
- Application code must not use SQLAlchemy 1.x `Query` API such as `db.query(Model)`.
- SQLAlchemy engines must use `pool_pre_ping=True`.
- SQL echo must remain disabled; SQL visibility must go through configured logging.
- Application schema must not be managed with `Base.metadata.create_all()`.
- Tests use the migrated PostgreSQL test database and must not manage the schema with
  `Base.metadata.create_all()`.
- Schema changes must go through Alembic; see `docs/backend/migrations.md`.
- Derived UX state must be computed in services unless a specific ADR requires persistence.
- Soft-delete behavior must be explicit and consistently filtered where applicable.
- Soft-deletable models use `deleted_at` and may use `deleted_by`; repositories must explicitly exclude deleted rows where relevant.
- Enum fields must remain synchronized with frontend Zod enum schemas.
- Python database enums should use `str, Enum` values and `pg_enum()` from `app/utils/enum_utils.py`.
- JSON-object columns must use `JSONB` from `sqlalchemy.dialects.postgresql`; generic `sqlalchemy.JSON` and `with_variant(...)` are not allowed.
- Values stored in a `JSONB` column must be persisted as dict/list objects, not `json.dumps` strings.
- Server defaults that exist in a migration must also be declared on the model; `alembic/env.py` runs with `compare_server_default=True`, so any asymmetry fails `alembic check`.
- Timestamp defaults should use `utcnow` from `app/utils/time_utils.py`.
- Indexes and uniqueness constraints belong in model `__table_args__`.
- Database access must stay in repositories.
- Business decisions must not be hidden inside models or repositories.
