## Scope
This file owns only:
- Historical content from the legacy health module README, preserved for reference.

This file must not contain:
- Current canonical behavior (see docs/domains/health.md).

Source of truth: historical

# Health — Legacy Archive

Canonical domain doc: `docs/domains/health.md`.

This archive preserves historically useful context from `backend/app/health/README.md` before that file was reduced to a pointer on 2026-05-29.

## From `backend/app/health/README.md` (archived 2026-05-29)

### Stable intent that remained useful

- The module exists for application health/readiness checks used by runtime monitoring and deployment verification.
- The domain does not own persistent database models.
- The endpoints are intentionally unauthenticated and map healthy/unhealthy state to `200`/`503`.
- The layering is API -> service -> repository, with the repository responsible for the DB connectivity probe.

### Historical test references

- `tests/health/api/test_health.py`
- `tests/health/repository/test_health_repository.py`
- `tests/health/service/test_health_service.py`

### Drift in the archived README

- The archived README documented only `GET /health`; current code also exposes `GET /ready`.
- The archived README said the repository used `db.query(1).first()`, but current code uses `self.db.execute(select(1))`.
