## Scope
This file owns only:
- Canonical local commands for running backend tooling.

This file must not contain:
- Architecture rules, product/domain behavior, or full verification criteria.

Source of truth: mandatory

# Backend Commands

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

Backend quality checks (individual):

```bash
./.venv/bin/ruff check .           # lint
./.venv/bin/ruff format --check .  # formatting check
./.venv/bin/pyright                # type/import/name checks
./.venv/bin/vulture                # unused code candidates
```

Backend fix commands:

```bash
./.venv/bin/ruff check . --fix     # autofix supported lint findings
./.venv/bin/ruff format .          # apply formatting
```

`pyright` and `vulture` do not provide safe automatic project fixes; review and edit reported findings manually.

Seed backend development data:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python scripts/seed_fake_data.py --reset
```

Export backend OpenAPI for contract review:

```bash
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.export_openapi --output openapi.json
```
