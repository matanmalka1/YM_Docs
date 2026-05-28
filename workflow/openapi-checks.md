## Scope
This file owns only:
- Workflow for checking OpenAPI/API contract consistency.

This file must not contain:
- Full API contract rules, backend architecture rules, or frontend component rules.

Source of truth: reference

# OpenAPI Checks

- Use OpenAPI checks when API request/response contracts, routes, schemas, or auth behavior change.
- Start the backend with the repo virtualenv before inspecting docs:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python -m app.main
```

- Inspect `http://localhost:8000/docs` or the generated OpenAPI JSON.
- Confirm route paths, methods, status codes, request schemas, response schemas, and auth requirements.
- If frontend contracts exist for the changed endpoint, update or verify them in the same task when in scope.
- Do not treat OpenAPI as a replacement for tests.
