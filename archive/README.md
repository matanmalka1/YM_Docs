## Scope
This file owns only:
- Archive policy for obsolete or historical documentation.

This file must not contain:
- Active architecture rules, product/domain rules, or TODOs.

Source of truth: reference

# Archive

All files in this directory are **historical / non-canonical**. None is a source of truth.

## Index of archived files

Every file currently under `docs/archive/` (excluding this README), with its current replacement.

### Domain legacy docs (pre-`docs/domains/` migration)

Each captured the old `backend/app/<domain>/README.md` (or equivalent legacy spec) before the canonical per-domain docs existed.

- `actions-legacy.md` — replaced by `docs/domains/actions.md`.
- `advance-payments-legacy.md` — replaced by `docs/domains/advance-payments.md`.
- `annual-reports-legacy.md` — replaced by `docs/domains/annual-reports.md`.
- `binders-legacy.md` — replaced by `docs/domains/binders.md`.
- `clients-legacy.md` — replaced by `docs/domains/clients.md`.
- `health-legacy.md` — replaced by `docs/domains/health.md`.
- `invoice-legacy.md` — replaced by `docs/domains/invoices.md`.
- `notifications-legacy.md` — replaced by `docs/domains/notifications.md`.
- `signature-requests-legacy.md` — replaced by `docs/domains/signature-requests.md`.
- `timeline-legacy.md` — replaced by `docs/domains/timeline.md`.
- `vat-reports-legacy.md` — replaced by `docs/domains/vat.md`.
- `work-queue-legacy.md` — replaced by `docs/domains/work-queue.md`.

### Historical TODOs and planning docs

- `api-todo-done.md` — historical completed API backlog. Current active backlog: `docs/api-todo.md`; current API rules: `docs/architecture/api-contracts.md`.
- `annual-reports-todo.md` — historical annual-reports task list. Current annual-report behavior: `docs/domains/annual-reports.md`; live backlog: `docs/api-todo.md`.
- `frontend-alignment-todo.md` — historical frontend/backend alignment task list. Archived after remaining items were resolved or no longer active. Current frontend rules: `docs/frontend/*`; current backend/API backlog: `docs/api-todo.md`.
- `frontend-page-convergence-log.md` — historical frontend page-convergence changelog. Current frontend page rules: `docs/frontend/page-structure.md`; current refactor status: `docs/frontend/page-refactor-status.md`.
- `migration-phases.md` — historical backend domain migration execution plan (structural migration complete). Replaced by `docs/project/backend-module-map.md` (package inventory) and `docs/domains/*` (canonical behavior).
- `frontend-ui-audit-todo.md` — frontend UI audit (16 findings: RTL, a11y, state/table, forms). All findings resolved in code before archiving; durable rules live in `docs/frontend/ui-guidelines.md`. Historical only.
- `design-system-audit-plan.md` — 2026-06-28 design-system reuse & primitive-API audit (full catalog F1–F19). Batch 1 (safe) + Batch 2 (additive props) done in code; conventions landed in `docs/frontend/ui-guidelines.md`. Remaining Batch 3 follow-up tracked at `docs/frontend/design-system-followup.md`. This catalog is historical only.
- `audit-refactor-progress.md` — phase-by-phase execution log (Phases 0–10) of the audit-model refactor that collapsed seven audit models to two (`EntityAuditLog` + `UserAuditLog`). Refactor complete. Current canonical audit behavior: `docs/domains/audit.md`. Historical only.
- `audit-refactor-phase-0-report.md` — Phase-0 baseline/inventory for that refactor (write-site counts, enum ownership, action/authorization matrices). Inputs to the now-complete refactor; superseded by the shipped code and `docs/domains/audit.md`. The implementation-plan companion doc was deleted (not archived) when the work shipped. Historical only.

### Historical generated scans / gap analyses

- `actual-domains-v1.md` — generated scan of `backend/app/` endpoints/services/repositories. Replaced by `docs/project/backend-module-map.md` (current inventory) and `docs/domains/*` (behavior); archived because it drifted from code.
- `domain-docs-inventory-legacy.md` — historical pre-migration domain-doc ownership inventory. Current ownership map: `docs/project/documentation-map.md`; current domain index: `docs/domains/README.md`.
- `gap-target-vs-actual-v1.md` — target-vs-actual domain gap analysis and build-order plan. Replaced by current domain docs and `docs/project/backend-module-map.md`. It refers to a `target-domains-v1.md` planning file that no longer exists in this repo; that reference is historical only and has No current canonical replacement identified.
- `raw-primitives-findings.md` — one-time grep-based audit (2026-06-23) of raw native elements and design-system reinvention in `frontend/src`, with a per-file TODO checklist. All buckets were dispositioned/resolved before archiving. The durable, non-obvious learnings (which narrow primitives exist and the boundaries where raw markup is correct) were extracted into `docs/frontend/canonical-helpers.md` ("Display & layout primitives") and `docs/frontend/ui-guidelines.md` ("Design-system usage"); those are the current source of truth. The checklist itself is historical only.

## Policy

- Archived docs are historical only.
- Do not use archived docs as active source of truth unless a current mandatory doc explicitly points to them.
- When archiving a doc, add it to the index above with what replaced it and why it is no longer canonical. If no replacement is obvious, write "No current canonical replacement identified".
