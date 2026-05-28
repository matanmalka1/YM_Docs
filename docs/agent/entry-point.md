## Scope
This file owns only:
- The detailed project-wide agent entry point.
- Required reading order before code or documentation changes.
- Top-level non-negotiable rules that apply across backend and frontend work.

This file must not contain:
- Full backend, frontend, API, database, workflow, or product/domain documentation.
- Temporary task lists or module-specific implementation notes.

Source of truth: mandatory

# Agent Entry Point

Before changing code or documentation, read in this order:

1. `docs/agent/behavior.md`
2. `docs/agent/decision-making.md`
3. `docs/agent/source-of-truth.md`
4. `docs/project/documentation-map.md`
5. `docs/architecture/backend.md` if touching backend code
6. `docs/architecture/frontend.md` if touching frontend code
7. `docs/architecture/api-contracts.md` if changing API contracts
8. `docs/architecture/database.md` and `docs/architecture/migrations.md` if touching models or schema
9. `docs/architecture/security.md` if touching auth, permissions, sensitive data, browser auth, or ownership checks
10. `docs/architecture/observability.md` if touching logs, request tracing, error reporting, or monitoring
11. `docs/workflow/testing.md` and `docs/workflow/verification.md` before finishing

Non-negotiable rules:

- You must follow the source-of-truth file for the area you touch.
- You must not add legacy compatibility unless explicitly requested.
- You must not add aliases, wrappers, or compatibility layers just to preserve old code.
- You must not add hidden fallback behavior.
- You must not change domain behavior unless the relevant domain docs are updated in the same task.
- You must stop and report conflicts between the user request, code, and docs before editing affected files.
- You must run or report the relevant verification from `docs/workflow/verification.md` before finishing.
