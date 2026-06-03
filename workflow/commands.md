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

Run backend tests (scoped, preferred):

```bash
JWT_SECRET=test-secret ./.venv/bin/python -m pytest -q tests/<path>
```

Run the full backend test suite:

```bash
JWT_SECRET=test-secret ./.venv/bin/python -m pytest -q
```

Seed backend development data:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python scripts/seed_fake_data.py --reset
```

Export backend OpenAPI for contract review:

```bash
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.export_openapi --output ../openapi.json
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

Regenerate the frontend API drift baseline after exporting OpenAPI:

```bash
npx openapi-typescript ../openapi.json -o src/types/generated.ts
```

See `docs/workflow/api-drift-ci.md` for how the CI uses this generated file.
