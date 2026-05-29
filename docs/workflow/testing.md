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
- See `docs/workflow/commands.md` for the canonical backend and frontend test commands.
- `JWT_SECRET=test-secret` is required because backend settings validate it at import time.
- Backend tests set `APP_ENV=test` in test configuration.
- Backend tests use SQLite in-memory with `StaticPool` and `check_same_thread=False`.
- Backend tests do not require Postgres.
- Backend test fixtures may use `Base.metadata.create_all()` for isolated test databases and must drop the schema after each test function.
- The standard backend DB fixture is `test_db`.
- Standard backend auth fixtures include `test_user`, `secretary_user`, `auth_token`, `secretary_token`, `advisor_headers`, and `secretary_headers`.
- The standard HTTP integration fixture is `client`; it overrides `get_db` to use `test_db`.
- Tests should use identity helpers such as `seed_client_identity()` and `seed_business()` to create the `LegalEntity` -> `ClientRecord` -> `Business` graph.
- Prefer service tests for business logic. Service tests instantiate the service directly with `test_db` and should not mock repositories or sessions.
- HTTP integration tests should use `TestClient` for routing, auth, request/response contracts, and middleware behavior.

- Backend business logic changes must include service-layer tests or a final explanation of why service-layer tests were not added.
- HTTP integration tests must cover routing, auth, request/response contracts, or middleware behavior when the changed surface is not already covered by existing tests.
- Do not mock the database for repository behavior that depends on ORM queries.
- Do not mock `Session` or repository methods in service tests that are meant to verify query or orchestration behavior.
- Backend test files should live under `tests/<domain>/service/` or `tests/<domain>/api/` where practical.
- Test functions should use `test_<what>_<condition>` names; private setup helpers inside test files should be prefixed with `_`.
- If frontend test coverage is missing for an area, report that honestly instead of inventing test results.
