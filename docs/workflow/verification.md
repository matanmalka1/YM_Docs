## Scope
This file owns only:
- Required verification steps before an agent says work is complete.

This file must not contain:
- Permanent architecture rules, test-writing guidance, or domain acceptance criteria.

Source of truth: mandatory

# Verification

Before finishing, verify the smallest meaningful scope that proves the change.

- For backend code changes, run relevant pytest targets with `./.venv/bin/python`.
- For backend model/schema changes, generate and review Alembic migrations.
- For API contract changes, verify OpenAPI and frontend contract impact.
- For frontend code changes, run relevant frontend checks such as typecheck, lint, build, tests, or architecture checks.
- For structural frontend changes, run `npm run arch:check`; run `npm run arch:check:strict` when touching cross-feature imports or barrels.
- For visible UI changes, verify the rendered UI in a browser when practical.
- For docs-only changes, manually verify file structure, links, paths, and scope boundaries.
- Report any checks that could not be run.
