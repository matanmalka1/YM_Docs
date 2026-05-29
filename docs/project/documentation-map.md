## Scope
This file owns only:
- The map of documentation areas and where agents look.
- References to existing domain docs without normalizing them.

This file must not contain:
- Full rules copied from other docs.
- Domain documentation rewrites or migrations.

Source of truth: reference

# Documentation Map

Paths in this file are relative to the root of the project docs repo or monorepo.

Start here:

- `AGENTS.md`
- `docs/agent/entry-point.md`

Project-wide agent rules:

- `docs/agent/entry-point.md`
- `docs/agent/behavior.md`
- `docs/agent/decision-making.md`
- `docs/agent/source-of-truth.md`

Project reference:

- `docs/project/overview.md`
- `docs/project/backend-module-map.md`
- `docs/project/documentation-map.md`
- `docs/project/domain-docs-inventory.md`

Architecture rules:

- `docs/architecture/backend.md`
- `docs/architecture/frontend.md`
- `docs/architecture/api-contracts.md`
- `docs/architecture/database.md`
- `docs/architecture/migrations.md`
- `docs/architecture/security.md`
- `docs/architecture/observability.md`

Workflow rules:

- `docs/workflow/commands.md`
- `docs/workflow/local-env.md`
- `docs/workflow/testing.md`
- `docs/workflow/verification.md`
- `docs/workflow/openapi-checks.md`
- `docs/workflow/git.md`

Existing domain docs not normalized in this phase:

- `backend/docs/annual_reports/`
- `backend/docs/vat_report/`
- `backend/docs/backend/domains/work-queue.md`
- `backend/docs/advance_payments_spec.md`
- `backend/docs/binder_lifecycle_refactor_spec.md`
- `backend/docs/frontend_screen_spec.md`
- `backend/app/*/README.md`

Do not migrate, rewrite, split, or normalize domain-specific docs in the generic documentation phase.
