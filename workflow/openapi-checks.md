## Scope
This file owns only:
- Workflow for checking OpenAPI/API contract consistency.

This file must not contain:
- Full API contract rules, backend architecture rules, or frontend component rules.

Source of truth: reference

# OpenAPI Checks

- Use OpenAPI checks when API request/response contracts, routes, schemas, or auth behavior change.
- For backend-to-frontend schema drift detection, see `docs/workflow/api-drift-ci.md`.
- Start the backend with the repo virtualenv before inspecting docs:

```bash
APP_ENV=development ENV_FILE=.env.development ./.venv/bin/python -m app.main
```

- Inspect `http://localhost:8000/docs` or the generated OpenAPI JSON.
- Confirm route paths, methods, status codes, request schemas, response schemas, and auth requirements.
- If frontend contracts exist for the changed endpoint, update or verify them in the same task when in scope.
- When a backend schema changes, regenerate/review the frontend `generated.ts` baseline and update manual frontend contracts for the affected feature. The generated file is a drift detector, not an application import.
- If OpenAPI output does not match code, report the mismatch before changing frontend contracts.
- Do not treat OpenAPI as a replacement for tests.
