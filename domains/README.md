## Scope
This file owns only:
- The index of canonical per-domain docs under `docs/domains/`.

This file must not contain:
- Domain rules, endpoints, or model details (those live in each domain doc).

Source of truth: reference

# Domain Docs Index

Canonical, code-verified docs — one per backend domain. Authoring template + rules: `docs/workflow/domain-docs-authoring.md`. Status tracker: `docs/project/domain-docs-plan.md`. Code findings surfaced during authoring live in each domain doc's `Known issues` section.

## Tax chain

- [advance-payments](advance-payments.md) — periodic income-tax prepayments (מקדמות), status, amounts, turnover snapshots.
- [annual-reports](annual-reports.md) — annual income-tax reports per client/year; lifecycle, schedules, financials, readiness, PDF export.
- [vat-reports](vat-reports.md) — VAT work items, invoices, filing, audit, summaries, exports.
- [tax-calendar](tax-calendar.md) — shared regulatory tax-calendar entries, deadline rules, materialization.
- [charge](charge.md) — office billing charges; draft/paid/canceled lifecycle + bulk actions.

## Client lifecycle & operations

- [clients](clients.md) — LegalEntity + ClientRecord identity, status, onboarding side-effects.
- [businesses](businesses.md) — businesses under a client; soft-delete/restore lifecycle; status-card aggregation.
- [binders](binders.md) — physical binder lifecycle: intake, location/capacity state machine, handover, audit.
- [correspondence](correspondence.md) — CRUD log of client interactions (calls, letters, email, meetings, fax).
- [notes](notes.md) — entity-scoped notes for client and business.
- [permanent-documents](permanent-documents.md) — durable per-client/business file storage with versioning, soft delete, signals.
- [authority-contact](authority-contact.md) — named contacts at government authorities scoped to a client record.
- [signature-requests](signature-requests.md) — digital signature request lifecycle: creation, signer token flow, audit.

## Work & messaging

- [tasks](tasks.md) — manual office tasks: standalone or source-linked work items with CRUD and lifecycle.
- [reminders](reminders.md) — scheduled reminder records (see Known issues — execution path currently broken).
- [notifications](notifications.md) — notification audit, preview/send, automatic binder-handover delivery.

## Aggregation & read surfaces

- [work-queue](work-queue.md) — aggregated open-work feed across tax/billing/binder sources + tasks.
- [timeline](timeline.md) — unified per-client operational activity feed (reverse-chronological).
- [dashboard](dashboard.md) — read-only overview and tax-submissions widgets; no persistent tables.
- [reports](reports.md) — read-only management reports: aging, VAT compliance, advance-payments, annual-report status.
- [search](search.md) — unified search across clients, active binders, and permanent documents.

## Cross-cutting & infra

- [audit](audit.md) — generic append-only entity audit trail (one read API, one log table).
- [users](users.md) — authentication, JWT lifecycle, user management, audit logs, password reset.
- [health](health.md) — `/health` + `/ready` liveness/readiness probes (no `/api/v1` prefix).
