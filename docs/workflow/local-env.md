## Scope
This file owns only:
- Local development environment conventions.

This file must not contain:
- Production deployment runbooks, architecture decisions, or product/domain behavior.

Source of truth: mandatory

# Local Environment

- Backend local work must use the backend repo virtualenv.
- Do not use global `python`, `python3`, or `pip` for backend work.
- Backend development command:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python -m app.main
```

- Backend API docs are available at `http://localhost:8000/docs` when the backend is running.
- Frontend local development must use `npm run dev` from `frontend/`.
- The frontend API base defaults to the local backend unless `VITE_API_BASE_URL` overrides it.
- Local environment changes must not require production secrets.
- Local development and test storage must not write outside project-owned local storage paths unless explicitly approved.
