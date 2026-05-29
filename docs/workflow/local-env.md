## Scope
This file owns only:
- Local development environment conventions.

This file must not contain:
- Production deployment runbooks, architecture decisions, or product/domain behavior.

Source of truth: mandatory

# Local Environment

- Backend local work must use the backend repo virtualenv; see `docs/workflow/commands.md` for the canonical commands.
- Backend local environment variables live in `.env.development`, which must not be committed.
- `DATABASE_URL` and `JWT_SECRET` are required for backend development.
- `CORS_ALLOWED_ORIGINS` defaults to localhost and 127.0.0.1 development ports.
- `LOG_SQL` is auto-enabled in development unless overridden.
- `LOG_LEVEL` defaults to `INFO`.
- Backend API docs are available at `http://localhost:8000/docs` when the backend is running.
- Backend development defaults to PostgreSQL, local filesystem storage under `./storage/`, and text logs.
- Backend tests default to SQLite, local filesystem storage, and text logs.
- In development and test, local storage is exposed at `/local-storage/*`.
- Production must not expose local filesystem storage; files are served from Cloudflare R2.
- Frontend local development must use `npm run dev` from `frontend/`.
- The frontend API base defaults to the local backend unless `VITE_API_BASE_URL` overrides it.
- Local environment changes must not require production secrets.
- Local development and test storage must not write outside project-owned local storage paths unless explicitly approved.
- Local storage is selected through `get_storage_provider()`; development and test use `LocalStorageProvider`, while staging and production use `S3StorageProvider`.
- Development startup runs `run_development_tax_calendar_bootstrap()` to seed default deadline rules when missing.
- Production must seed tax calendar data through explicit seeding or migration workflow, not development bootstrap.
- Development starts `daily_expiry_job()` from lifespan; tests patch this job to avoid side effects.
- `NOTIFICATIONS_ENABLED` defaults to `false`; disabled email delivery logs and returns success without sending.
