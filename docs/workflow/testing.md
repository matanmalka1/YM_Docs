## Scope
This file owns only:
- How tests are written and run.

This file must not contain:
- Broad completion criteria, manual QA checklist, permanent architecture rules, or domain acceptance criteria.

Source of truth: mandatory

# Testing

- Run the most relevant tests for the files changed by default.
- Run the full suite when the change is broad, shared, risky, or explicitly requested.
- Do not claim tests passed unless they were run in the current task.
- Backend tests are run from `backend/` with the repo virtualenv:

```bash
JWT_SECRET=test-secret ./.venv/bin/python -m pytest -q tests/<path>
```

- Backend business logic changes must include service-layer tests or a final explanation of why service-layer tests were not added.
- HTTP integration tests must cover routing, auth, request/response contracts, or middleware behavior when the changed surface is not already covered by existing tests.
- Do not mock the database for repository behavior that depends on ORM queries.
- Frontend tests are run from `frontend/`:

```bash
npm run test
```

- If frontend test coverage is missing for an area, report that honestly instead of inventing test results.
