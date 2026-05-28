## Scope
This file owns only:
- Required verification steps before an agent says work is complete.

This file must not contain:
- Permanent architecture rules, test-writing guidance, or domain acceptance criteria.

Source of truth: mandatory

# Verification

Before finishing, verify the smallest meaningful scope that proves the change.

- Backend code changes must run relevant pytest targets with `./.venv/bin/python`, or report why they were not run.
- Backend persistence model or database schema changes must include generated and reviewed Alembic migrations when the database schema changes.
- API contract changes must verify OpenAPI and frontend contract impact.
- Frontend code changes must run the relevant frontend checks: typecheck, lint, build, tests, or architecture checks.
- Structural frontend changes must run `npm run arch:check`; cross-feature import or barrel changes must run `npm run arch:check:strict`.
- Visible UI changes must be verified in a browser, or the final response must explain why browser verification was not run.
- Docs-only changes must manually verify file structure, links, paths, and scope boundaries.
- Report any checks that could not be run.
