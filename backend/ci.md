# Backend CI

## What it is

GitHub Actions workflow in `YM_Backend` that runs on every pull request and every push to `main`.

Five parallel jobs gate code quality, types, tests, schema sync, and database migrations.

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
  ┌────────┬──────────┬───────────┬──────────────┬─────────────────┐
  │  lint  │ typecheck│   test    │ openapi-sync │   migrations    │
  └────────┴──────────┴───────────┴──────────────┴─────────────────┘
     │         │           │             │               │
  ruff      pyright     pytest      check_contract   Postgres 17 service
  check     (lenient)   + coverage   _sync.py        + alembic roundtrip
  format                (SQLite)    (no DB)          + alembic check
```

| Job | Runs | Notes |
|-----|------|-------|
| **lint** | `ruff check .` + `ruff format --check .` | Installs only ruff (fast). `alembic/` is excluded per `pyproject.toml`. |
| **typecheck** | `pyright` | Lenient — `pyrightconfig.json` disables most `report*` checks, so it catches undefined names / bad imports / syntax, not strict typing. `--pythonpath` bypasses the config's absolute local `venvPath`. |
| **test** | `pytest --cov=app` | In-memory SQLite; sets `APP_ENV=test` itself; `JWT_SECRET` is the only required env. Coverage is reported, not enforced. |
| **openapi-sync** | `check_contract_sync.py` | Ensures committed `openapi.json` matches the app. No DB. See [Why openapi-sync matters](#why-openapi-sync-matters). |
| **migrations** | head-divergence + Postgres roundtrip + `alembic check` | Real Postgres 17 service (matches Neon in production). |

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

After the roundtrip the job runs `alembic check`, which autogenerates against the live SQLAlchemy metadata and fails if a model was changed without a matching migration. `alembic/env.py` sets `compare_type=True`, so column-type changes are caught alongside added/dropped tables and columns.

`compare_server_default` is intentionally **not** enabled: `client_records.office_client_number` carries a `nextval()` server default in the migration but not on the model — the model omits it so SQLite test DDL (`create_all`) does not emit `CREATE SEQUENCE`. Enabling the flag would flag that intentional asymmetry on every run.

---

## Why openapi-sync matters

The frontend [api-drift](../workflow/api-drift-ci.md) check pulls from the **committed** `openapi.json`, not from a freshly generated one. So if a backend schema changes but `openapi.json` is not regenerated and committed, the frontend drift check passes against a stale file — drift slips past both repos.

The `openapi-sync` job runs `scripts/tooling/check_contract_sync.py`, which regenerates the schema from the live app and compares it to the committed `openapi.json`. It fails if they differ. No database is needed.

Fix when it fails:

```bash
cd backend
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.export_openapi --output openapi.json
# commit the updated openapi.json
```

---

## Key files

| File | Role |
|------|------|
| `backend/.github/workflows/ci.yml` | the workflow |
| `backend/pyproject.toml` | ruff config; `target-version = "py314"`; `alembic` excluded from lint |
| `backend/pyrightconfig.json` | pyright config; absolute local `venvPath` (bypassed in CI via `--pythonpath`); most `report*` checks disabled |
| `backend/scripts/tooling/check_contract_sync.py` | compares committed `openapi.json` to the live app |
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

Python is pinned to `3.14` across all jobs, matching the backend target and the frontend [api-drift](../workflow/api-drift-ci.md) job.

---

## Running the same checks locally

```bash
cd backend

# lint
./.venv/bin/ruff check .
./.venv/bin/ruff format --check .

# typecheck
./.venv/bin/pyright

# tests (+ coverage)
APP_ENV=test JWT_SECRET=x ./.venv/bin/pytest --cov=app --cov-report=term-missing

# openapi.json in sync with the app
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.check_contract_sync

# migration roundtrip + alembic check against a local Postgres
createdb ci_roundtrip_check
export APP_ENV=test JWT_SECRET=x
export DATABASE_URL="postgresql+psycopg2://$(whoami)@localhost:5432/ci_roundtrip_check"
./.venv/bin/alembic upgrade head
./.venv/bin/alembic downgrade base
./.venv/bin/alembic upgrade head    # must succeed
./.venv/bin/alembic check           # models match migrations
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
2. Require status checks: `lint`, `typecheck`, `test`, `openapi-sync`, `migrations`.

After that, a red job blocks merge.

---

## Related

- [api-drift-ci.md](../workflow/api-drift-ci.md) — the frontend-side CI that guards backend→frontend schema drift.
