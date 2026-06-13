# Backend CI

## What it is

GitHub Actions workflow in `YM_Backend` that runs on every pull request and every push to `main`.

Three parallel jobs gate code quality, tests, and database migrations.

Current mode: **run-but-don't-block** — results are reported but not yet enforced as required status checks. See [Making it a hard gate](#making-it-a-hard-gate).

---

## Why it exists

Before this, `YM_Backend` had no CI. Lint, tests, and migration validity depended on developer discipline.

The migrations job in particular catches a class of bug that local SQLite testing hides: Postgres-only DDL (enum types, `JSONB`, `ALTER TYPE`, gin/partial indexes, server defaults, deferred constraints) and broken `downgrade()` functions.

---

## Jobs

```
push to main / pull_request
     ↓
  ┌──────────┬──────────┬─────────────────┐
  │  lint    │  test    │   migrations    │
  └──────────┴──────────┴─────────────────┘
     │            │              │
  ruff check   pytest      Postgres 17 service
  ruff format  (SQLite     + alembic roundtrip
   --check      in-memory)  (upgrade→downgrade→upgrade)
```

| Job | Runs | Notes |
|-----|------|-------|
| **lint** | `ruff check .` + `ruff format --check .` | Installs only ruff (fast). `alembic/` is excluded per `pyproject.toml`. |
| **test** | `pytest` | Tests use in-memory SQLite and set `APP_ENV=test` themselves; `JWT_SECRET` is the only required env. |
| **migrations** | head-divergence check + Postgres roundtrip | Real Postgres 17 service (matches Neon in production). |

---

## Why the migrations job uses real Postgres

SQLite has no enum types and silently accepts Postgres-only DDL, so a SQLite-only migration check passes code that breaks in production.

The job runs the migrations against a `postgres:17` service container and performs a full roundtrip:

```
alembic upgrade head      # applies all migrations
alembic downgrade base    # reverses all migrations
alembic upgrade head      # re-applies — fails if downgrade left objects behind
```

The roundtrip is what catches broken `downgrade()` functions. It already caught one real bug: `0001_initial` left 50 Postgres enum types behind on downgrade, because `op.drop_table` does not drop enum types. See [[project-alembic-enum-downgrade]] in agent memory and the fix in `alembic/versions/0001_initial.py`.

The head-divergence check (`alembic heads` must show exactly one head) needs no database and runs first.

---

## Key files

| File | Role |
|------|------|
| `backend/.github/workflows/ci.yml` | the workflow |
| `backend/pyproject.toml` | ruff config; `target-version = "py314"`; `alembic` excluded from lint |
| `backend/pytest.ini` | pytest config (SQLite, `APP_ENV=test`) |
| `backend/alembic/versions/*` | migrations validated by the roundtrip |
| `backend/app/config.py` | `APP_ENV=test` bypasses the localhost `DATABASE_URL` guard so CI can point alembic at the Postgres service |

---

## Environment

| Var | Value in CI | Why |
|-----|-------------|-----|
| `APP_ENV` | `test` | Bypasses the production localhost guard in `app/config.py`; lets tests use SQLite. |
| `JWT_SECRET` | `ci-test-only` | Required for app import; value is throwaway. |
| `DATABASE_URL` | `postgresql+psycopg2://postgres:postgres@localhost:5432/binder_crm` | Migrations job only — points alembic at the Postgres service. Scheme must be `+psycopg2` (the installed driver). |

Python is pinned to `3.14` across all jobs, matching the backend target and the frontend [api-drift](api-drift-ci.md) job.

---

## Running the same checks locally

```bash
cd backend

# lint
./.venv/bin/ruff check .
./.venv/bin/ruff format --check .

# tests
APP_ENV=test JWT_SECRET=x ./.venv/bin/pytest

# migration roundtrip against a local Postgres
createdb ci_roundtrip_check
export APP_ENV=test JWT_SECRET=x
export DATABASE_URL="postgresql+psycopg2://$(whoami)@localhost:5432/ci_roundtrip_check"
./.venv/bin/alembic upgrade head
./.venv/bin/alembic downgrade base
./.venv/bin/alembic upgrade head    # must succeed
dropdb ci_roundtrip_check
```

---

## Adding a new enum migration

Any new migration that introduces a `sa.Enum(..., name='x')` column must drop that type in `downgrade()`, or the roundtrip job will fail:

```python
def downgrade() -> None:
    op.drop_table('...')
    if op.get_bind().dialect.name == 'postgresql':
        op.execute("DROP TYPE IF EXISTS x")
```

Postgres enum types are standalone objects; `drop_table` does not remove them.

---

## Making it a hard gate

The workflow runs but does not block merges yet. To enforce:

1. GitHub → repo **Settings → Branches → Branch protection rules** for `main`.
2. Require status checks: `lint`, `test`, `migrations`.

After that, a red job blocks merge.

---

## Related

- [api-drift-ci.md](api-drift-ci.md) — the frontend-side CI that guards backend→frontend schema drift.
