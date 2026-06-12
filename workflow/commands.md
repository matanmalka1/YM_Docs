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
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.export_openapi --output openapi.json
```

Run frontend commands from `frontend/`:

Dev / build:

```bash
npm run dev          # vite dev server
npm run dev:lan      # vite dev server on 0.0.0.0 (LAN access)
npm run build        # typecheck then vite build
npm run preview      # serve the production build locally
```

Quality checks (individual):

```bash
npm run typecheck         # tsc --noEmit on tsconfig.app.json
npm run gen:types         # regenerate src/types/generated.ts from ../backend/openapi.json
npm run lint              # eslint, fails on any warning
npm run lint:fix          # eslint with autofix
npm run format            # prettier write
npm run format:check      # prettier check (no write)
npm run arch:check        # architecture boundary check
npm run arch:check:strict # architecture check, strict mode
npm run test              # vitest run
npm run unused            # knip: unused files/exports/deps
npm run jscpd             # copy-paste detection
npm run jscpd:strict      # copy-paste detection, zero-tolerance threshold
```

Aggregate gates:

```bash
npm run check         # typecheck + lint + format:check + arch:check + test + unused
npm run check:strict  # check with strict arch + jscpd:strict
npm run fix           # lint:fix + format
```

Regenerate the frontend API drift baseline after exporting OpenAPI:

```bash
npm run gen:types
```

See `docs/workflow/api-drift-ci.md` for how the CI uses this generated file.
