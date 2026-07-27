## Scope
This file owns only:
- How backend tests are written and run.

This file must not contain:
- Broad completion criteria, manual QA checklist, permanent architecture rules, or domain acceptance criteria.

Source of truth: mandatory

# Backend Testing

- Run the most relevant tests for the files changed by default.
- Run the full suite when the change is broad, shared, risky, or explicitly requested.
- Do not claim tests passed unless they were run in the current task.
- See `docs/backend/commands.md` for the canonical backend test commands.
- `JWT_SECRET=test-secret` is required because backend settings validate it at import time.
- Backend tests set `APP_ENV=test` in test configuration.
- Backend tests use the dedicated PostgreSQL database `binder_crm_test`.
- Local tests require the Docker Compose `db_test` service on port 5433.
- CI starts PostgreSQL and applies Alembic migrations before running pytest. Locally the migration
  step is manual and is not run by `conftest.py`; see `docs/backend/commands.md`.
- Each test runs inside an outer transaction that is rolled back by the `test_db` fixture.
- The standard backend DB fixture is `test_db`.
- Standard backend auth fixtures include `test_user`, `secretary_user`, `auth_token`, `secretary_token`, `advisor_headers`, and `secretary_headers`.
- The standard HTTP integration fixture is `client`; it overrides `get_db` to use `test_db`.
- Parameterized test-data fixtures are `user_factory`, `client_factory`, `business_factory`,
  `create_client_with_business`, `annual_report_factory`, `annual_report_model_factory`,
  `binder_factory`, `binder_intake_factory`, `binder_intake_material_factory`, `charge_factory`,
  `task_factory`, `advance_payment_factory`, `permanent_document_factory`,
  `signature_request_factory`, `notification_factory`, `reminder_factory`, `invoice_factory`,
  `vat_work_item_factory`, `tax_calendar_entry_factory`, and `authority_contact_factory`. Prefer
  them over repeating ORM constructors or adding equivalent per-file setup helpers.
- Factory defaults create unique identities. Pass only fields relevant to the behavior under test.
- `user_factory` and `create_client_with_business` commit by default because their common consumers
  are API tests. The other factories flush by default; pass `commit=True` only when a commit boundary
  is part of the setup requirement.
- Use the fixture, never the factory class. Importing `tests.factories` inside a test module and
  instantiating a factory bypasses the fixture contract and its per-test sequencing.
- Factories that own a required foreign key accept the parent as either object or id
  (`client=` / `client_record_id=`, `business=` / `business_id=`); passing both raises. They
  auto-create the parent only when the foreign key is NOT NULL and nothing was passed.
- `client_factory(legal_entity_id=...)` attaches a new `ClientRecord` to an existing `LegalEntity`.
  In that mode only `ClientRecord` fields are accepted; `LegalEntity`/`Person` fields raise.
- `business_factory` accepts `deleted_at` and `deleted_by` for repository tests that exercise
  soft-deleted business rows.
- The identity helpers `seed_client_identity()` and `seed_business()` are the implementation behind
  `client_factory` / `business_factory` / `create_client_with_business`. Test modules should call the
  fixtures, not these helpers directly.
- Build entities by hand only when the database structure itself is the subject of the test:
  constraint and foreign-key violations, server-default and nullability guards, deliberately invalid
  rows, and states no factory can express. Say so in the test when it is not obvious.
- A module-local setup helper is acceptable only when it carries test-specific data beyond what the
  factory provides, and it must take the fixture as a parameter rather than a `Session`.
- Prefer service tests for business logic. Service tests instantiate the service directly with `test_db` and should not mock repositories or sessions.
- HTTP integration tests should use `TestClient` for routing, auth, request/response contracts, and middleware behavior.

- Backend business logic changes must include service-layer tests or a final explanation of why service-layer tests were not added.
- HTTP integration tests must cover routing, auth, request/response contracts, or middleware behavior when the changed surface is not already covered by existing tests.
- Do not mock the database for repository behavior that depends on ORM queries.
- Do not mock `Session` or repository methods in service tests that are meant to verify query or orchestration behavior.
- Backend test files should live under `tests/<domain>/service/` or `tests/<domain>/api/` where practical.
- Test functions should use `test_<what>_<condition>` names; private setup helpers inside test files should be prefixed with `_`.
- Backend per-file test coverage should stay at or above 90%; monitor for regressions in CI.
