## Scope
This file owns only:
- Canonical local commands for running project tooling.

This file must not contain:
- Architecture rules, product/domain behavior, or full verification criteria.

Source of truth: mandatory

# Commands

- Run backend commands from `backend/`.
- Backend Python commands must use `./.venv/bin/python` or `./.venv/bin/pip`.
- Do not use global `python`, `python3`, or `pip` for backend work.

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

Run frontend commands from `frontend/`:

```bash
npm run dev
npm run build
npm run typecheck
npm run lint
npm run arch:check
npm run arch:check:strict
npm run test
```
