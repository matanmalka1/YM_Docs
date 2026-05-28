## Scope
This file owns only:
- How tests are written and run.

This file must not contain:
- Broad completion criteria, manual QA checklist, permanent architecture rules, or domain acceptance criteria.

Source of truth: mandatory

# Testing

- Run the most relevant tests for the files changed by default.
- Run the full suite when the change is broad, shared, risky, or explicitly requested.
- Backend tests are run from `backend/` with the repo virtualenv:

```bash
JWT_SECRET=test-secret ./.venv/bin/python -m pytest -q tests/<path>
```

- Backend service tests are preferred for business logic.
- HTTP integration tests are appropriate for routing, auth, request/response contracts, and middleware behavior.
- Do not mock the database for repository behavior that depends on ORM queries.
- Frontend tests are run from `frontend/`:

```bash
npm run test
```

- If frontend test coverage is missing for an area, report that honestly instead of inventing test results.
