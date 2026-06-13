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

Mandatory starting point. Read this before changing any code or documentation.

## Always read first

1. `docs/agent/behavior.md`
2. `docs/agent/decision-making.md`
3. `docs/agent/source-of-truth.md`
4. `docs/project/documentation-map.md`

Reference (not mandatory): `docs/project/glossary.md` — terminology disambiguation, e.g. `client_id` vs `client_record_id`.

When reviewing a code or docs change before approval: `docs/agent/code-review-playbook.md`.

## Read before, by change type

### Backend

- **Backend change** → `docs/backend/architecture.md`
- **DB / migration change** → `docs/backend/database.md` + `docs/backend/migrations.md`
- **Backend error-code / OpenAPI error-doc change** → `docs/backend/error-codes.md` + `docs/backend/error-doc-matrix.md`
- **Backend tests / verification** (before finishing) → `docs/backend/testing.md` + `docs/workflow/verification.md`

### Frontend

- **Frontend change** → `docs/frontend/architecture.md` + `docs/frontend/page-structure.md`
- **Frontend UI / styling / interaction change** → also read `docs/frontend/ui-guidelines.md`
- **Frontend tests / verification** (before finishing) → `docs/frontend/testing.md` + `docs/workflow/verification.md`

### Cross-cutting (both stacks)

- **API contract change** → `docs/architecture/api-contracts.md` (binding); `docs/architecture/api-contract-standard.md` for worked examples
- **Auth / security change** → `docs/architecture/security.md`
- **Observability change** (logs, tracing, error reporting) → `docs/architecture/observability.md`

## Conflict priority

When sources disagree, the higher item wins:

1. ADRs (`docs/adr/*`)
2. Architecture docs (`docs/architecture/*`, `docs/backend/*`, `docs/frontend/*`)
3. Workflow docs (`docs/workflow/*`)
4. Project docs (`docs/project/*`)
5. Existing code

Stop and report any conflict between the user request, code, and docs before editing affected files.

- Do not infer architecture from messy existing code when docs define stricter rules. Docs win; follow them.
- A docs file is source of truth only if it is listed in `docs/project/documentation-map.md`.

## Non-negotiable rules

- You must follow the source-of-truth file for the area you touch.
- You must not add legacy compatibility unless explicitly requested.
- You must not add aliases, wrappers, or compatibility layers just to preserve old code.
- You must not add hidden fallback behavior.
- You must not change domain behavior unless the relevant domain docs are updated in the same task.
- You must stop and report conflicts between the user request, code, and docs before editing affected files.
- You must run or report the relevant verification from `docs/workflow/verification.md` before finishing.
