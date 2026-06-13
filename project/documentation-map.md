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
- `docs/architecture/update-request-conventions.md` — PATCH/`*UpdateRequest` conventions: empty-PATCH 422, unknown-field rejection, explicit-null semantics, single-payload exceptions, `ClientUpdateRequest` field ownership (non-normative; defers to `api-contracts.md`).
- `docs/architecture/error-codes.md` — registry of `DOMAIN.REASON` error-code namespaces + status mapping (non-normative).
- `docs/architecture/error-doc-matrix.md` — per-endpoint OpenAPI error-status documentation matrix (#53), with raise-site evidence (non-normative; defers to `api-contracts.md`).
- `docs/architecture/database.md` ★ — database modeling rules.
- `docs/architecture/migrations.md` ★ — schema migration rules.
- `docs/architecture/security.md` ★ — auth, token, and security rules.
- `docs/architecture/observability.md` ★ — logging and observability rules.

## Workflow

- `docs/workflow/commands.md` ★ — standard project commands.
- `docs/workflow/local-env.md` ★ — local environment setup.
- `docs/workflow/testing.md` ★ — testing rules.
- `docs/workflow/verification.md` ★ — change verification rules.
- `docs/workflow/domain-docs-authoring.md` ★ — reusable template + parallel rules for writing canonical per-domain docs.
- `docs/workflow/git.md` ★ — git workflow rules.
- `docs/workflow/openapi-checks.md` — OpenAPI check usage.
- `docs/workflow/api-drift-ci.md` — GitHub Actions drift check for backend OpenAPI vs frontend generated type baseline.
- `docs/workflow/backend-ci.md` — GitHub Actions backend CI: ruff lint, pyright, pytest+coverage, openapi.json sync, alembic migration roundtrip + check against Postgres 17.
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

## Domains (canonical)

Per-domain canonical docs, one file each. Authoring rules: `docs/workflow/domain-docs-authoring.md`.

- `docs/domains/README.md` ★ — index of all 30 domain docs.

Tax chain:

- `docs/domains/advance-payments.md` ★
- `docs/domains/annual-reports.md` ★
- `docs/domains/vat.md` ★
- `docs/domains/tax-calendar.md` ★
- `docs/domains/charges.md` ★
- `docs/domains/invoices.md` ★

Client lifecycle & operations:

- `docs/domains/clients.md` ★
- `docs/domains/legal-entities.md` ★
- `docs/domains/businesses.md` ★
- `docs/domains/contacts.md` ★
- `docs/domains/binders.md` ★
- `docs/domains/communications.md` ★
- `docs/domains/notes.md` ★
- `docs/domains/documents.md` ★
- `docs/domains/permanent-documents.md` ★
- `docs/domains/authority-contacts.md` ★
- `docs/domains/signature-requests.md` ★

Work & messaging:

- `docs/domains/tasks.md` ★
- `docs/domains/reminders.md` ★
- `docs/domains/notifications.md` ★
- `docs/domains/actions.md` ★

Aggregation & read:

- `docs/domains/work-queue.md` ★
- `docs/domains/alerts.md` ★
- `docs/domains/timeline.md` ★
- `docs/domains/dashboard.md` ★
- `docs/domains/reports.md` ★
- `docs/domains/search.md` ★

Cross-cutting & infra:

- `docs/domains/audit.md` ★
- `docs/domains/users.md` ★
- `docs/domains/health.md` ★

## Flows (cross-domain)

Cross-domain flow documentation. Each file traces one flow from trigger to side effects, based on actual code.

- `docs/flows/README.md` — index of all flow docs.
- `docs/flows/01-client-business-creation-cascade.md` — client + business creation, onboarding orchestration.
- `docs/flows/02-annual-report-status-transition.md` — status machine, signature request lifecycle.
- `docs/flows/03-client-freeze-close-cascade.md` — cascade effects.
- `docs/flows/04-binder-material-intake.md` — intake, binder resolution, VAT auto-advance.
- `docs/flows/05-vat-work-item-creation.md` — work item creation, tax calendar materialization.
- `docs/flows/06-charge-work-queue.md` — charge-to-work-queue derivation.
- `docs/flows/07-client-status-card.md` — cross-domain client status aggregation.
- `docs/flows/08-dashboard-overview.md` — dashboard overview, role-based access.
- `docs/flows/09-work-queue-assembly.md` — full work queue build, task merging, filters.

## Legacy backend domain docs (pointers only)

These files under `backend/docs/**` and `backend/app/*/README.md` have been replaced with pointers to the canonical docs above; historical content lives in `docs/archive/`. Do not edit them as sources of truth.
