## Scope
This file owns only:
- The map of documentation areas and where agents look.
- References to existing domain docs without normalizing them.

This file must not contain:
- Full rules copied from other docs.
- Domain documentation rewrites or migrations.

Source of truth: reference

# Documentation Map

Official docs index and navigation map. Paths are relative to the root of the project docs repo. **★ = source of truth** (mandatory/binding).

## Precedence

- ADRs override all other docs when relevant.
- Architecture docs override workflow and project docs for technical rules.
- Archive docs are historical only and are not source of truth.

## Start here

- `AGENTS.md` — top-level entry pointer for agents.
- `docs/agent/entry-point.md` ★ — canonical project-wide agent instructions; read first.

## Project

- `docs/project/overview.md` — what the product is and its high-level shape.
- `docs/project/documentation-map.md` — this index/navigation map.
- `docs/project/backend-module-map.md` — where backend modules live.
- `docs/project/domain-docs-inventory.md` — list of existing domain-specific docs.
- `docs/project/config-reference.md` — index of backend env/config variables (source: `backend/app/config.py`).

## Architecture (technical source of truth)

- `docs/architecture/backend.md` ★ — backend layering and structure rules.
- `docs/architecture/frontend.md` ★ — frontend architecture rules.
- `docs/architecture/api-contracts.md` ★ — binding public API contract rules.
- `docs/architecture/api-contract-standard.md` — worked API examples (non-normative; defers to `api-contracts.md`).
- `docs/architecture/error-codes.md` — registry of `DOMAIN.REASON` error-code namespaces + status mapping (non-normative).
- `docs/architecture/database.md` ★ — database modeling rules.
- `docs/architecture/migrations.md` ★ — schema migration rules.
- `docs/architecture/security.md` ★ — auth, token, and security rules.
- `docs/architecture/observability.md` ★ — logging and observability rules.

## Workflow

- `docs/workflow/commands.md` ★ — standard project commands.
- `docs/workflow/local-env.md` ★ — local environment setup.
- `docs/workflow/testing.md` ★ — testing rules.
- `docs/workflow/verification.md` ★ — change verification rules.
- `docs/workflow/git.md` ★ — git workflow rules.
- `docs/workflow/openapi-checks.md` — OpenAPI check usage.
- `docs/workflow/local-mobile-testing.md` — local mobile testing guide.

## Agent

- `docs/agent/entry-point.md` ★ — canonical agent instructions.
- `docs/agent/behavior.md` ★ — required agent behavior.
- `docs/agent/decision-making.md` ★ — how agents make decisions.
- `docs/agent/source-of-truth.md` ★ — how to resolve doc precedence.

## ADR (override all docs when relevant)

- `docs/adr/0001-no-legacy-compatibility.md` — no legacy compatibility by default.
- `docs/adr/0002-router-service-repository.md` — router/service/repository layering.
- `docs/adr/0003-no-hidden-fallbacks.md` — no hidden fallbacks.
- `docs/adr/0004-project-docs-ownership.md` — project docs ownership model.

## Archive (historical only, not source of truth)

- `docs/archive/README.md` — index of retired/historical docs.

## Domain docs (not normalized in this phase)

- `backend/docs/backend/domains/annual_reports/`
- `backend/docs/backend/domains/vat_report/`
- `backend/docs/backend/domains/work-queue.md`
- `backend/docs/backend/domains/advance_payments_spec.md`
- `backend/docs/backend/domains/binder_lifecycle_refactor_spec.md`
- `backend/docs/frontend_screen_spec.md`
- `backend/app/*/README.md`

Do not migrate, rewrite, split, or normalize domain-specific docs in the generic documentation phase.
