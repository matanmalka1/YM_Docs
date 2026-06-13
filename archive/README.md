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

- `annual-reports-todo.md` — historical annual-reports task list. Current annual-report behavior: `docs/domains/annual-reports.md`; live backlog: `docs/api-todo.md`.
- `migration-phases.md` — historical backend domain migration execution plan (structural migration complete). Replaced by `docs/project/backend-module-map.md` (package inventory) and `docs/domains/*` (canonical behavior).

### Historical generated scans / gap analyses

- `actual-domains-v1.md` — generated scan of `backend/app/` endpoints/services/repositories. Replaced by `docs/project/backend-module-map.md` (current inventory) and `docs/domains/*` (behavior); archived because it drifted from code.
- `gap-target-vs-actual-v1.md` — target-vs-actual domain gap analysis and build-order plan. Replaced by current domain docs and `docs/project/backend-module-map.md`. It refers to a `target-domains-v1.md` planning file that no longer exists in this repo; that reference is historical only and has No current canonical replacement identified.

## Policy

- Archived docs are historical only.
- Do not use archived docs as active source of truth unless a current mandatory doc explicitly points to them.
- When archiving a doc, add it to the index above with what replaced it and why it is no longer canonical. If no replacement is obvious, write "No current canonical replacement identified".
