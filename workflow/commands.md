## Scope
This file owns only:
- Canonical local commands for running project tooling.

This file must not contain:
- Architecture rules, product/domain behavior, or full verification criteria.

Source of truth: mandatory

# Commands

Backend commands are run from `backend/` and must use the repo virtualenv.

Run backend:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python -m app.main
```

Install backend dependencies:

```bash
./.venv/bin/pip install -r requirements.txt
```

Run backend tests:

```bash
JWT_SECRET=test-secret ./.venv/bin/python -m pytest -q tests/<path>
```

Frontend commands are run from `frontend/`:

```bash
npm run dev
npm run build
npm run typecheck
npm run lint
npm run arch:check
npm run arch:check:strict
npm run test
```
