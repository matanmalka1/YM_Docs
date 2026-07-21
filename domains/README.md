## Scope
This file owns only:
- The index of canonical per-domain docs under `docs/domains/`.

This file must not contain:
- Domain rules, endpoints, or model details (those live in each domain doc).

Source of truth: reference

# Domain Docs Index

Canonical, code-verified docs — one per backend domain. Authoring template + rules: `docs/workflow/domain-docs-authoring.md`. Code findings surfaced during authoring live in each domain doc's `Known issues` section.

For terminology disambiguation (e.g. `client_id` vs `client_record_id`, AuditTrail vs Timeline), see `docs/project/glossary.md` — a pointer index, not a behavior source of truth.

## Tax chain

- [advance-payments](advance-payments.md) — periodic income-tax prepayments (מקדמות), status, amounts, turnover snapshots.
- [annual-reports](annual-reports.md) — annual income-tax reports per client/year; lifecycle, schedules, financials, readiness, PDF export.
- [vat](vat.md) — VAT work items, invoices, filing, audit, summaries, exports.
- [tax-calendar](tax-calendar.md) — shared regulatory tax-calendar entries, deadline rules, materialization.
- [charges](charges.md) — office billing charges; draft/paid/canceled lifecycle + bulk actions.
- [invoices](invoices.md) — external invoice references attached one-to-one to issued charges.

## Client lifecycle & operations

- [clients](clients.md) — ClientRecord CRM anchor, status, ownership, onboarding side-effects.
- [legal-entities](legal-entities.md) — legal/tax identity graph: LegalEntity, Person, and person-to-entity links.
- [businesses](businesses.md) — businesses under a client; soft-delete/restore lifecycle; status-card aggregation.
- [contacts](contacts.md) — scaffold-only future client-contact domain; no behavior implemented.
- [binders](binders.md) — physical binder lifecycle: intake, location/capacity state machine, handover, audit.
- [communications](communications.md) — CRUD log of client interactions (calls, letters, email, meetings, fax).
- [notes](notes.md) — entity-scoped notes for client and business.
- [documents](documents.md) — parent package for document domains; currently contains permanent documents.
- [permanent-documents](permanent-documents.md) — durable per-client/business file storage with versioning, soft delete, signals.
- [authority-contacts](authority-contacts.md) — named contacts at government authorities scoped to a client record.
- [signature-requests](signature-requests.md) — digital signature request lifecycle: creation, signer token flow, audit.

## Work & messaging

- [actions](actions.md) — backend action metadata factories and cross-domain obligation orchestration helpers.
- [tasks](tasks.md) — manual office tasks: standalone or source-linked work items with CRUD and lifecycle.
- [reminders](reminders.md) — scheduled reminder records (see Known issues — execution path currently broken).
- [notifications](notifications.md) — notification audit, preview/send, automatic binder-handover delivery.

## Aggregation & read surfaces

- [work-queue](work-queue.md) — aggregated open-work feed across tax/billing/binder sources + tasks.
- [alerts](alerts.md) — internal attention-signal calculation used by the dashboard attention board.
- [timeline](timeline.md) — unified per-client operational activity feed (reverse-chronological).
- [dashboard](dashboard.md) — read-only overview and tax-submissions widgets; no persistent tables.
- [reports](reports.md) — read-only management reports: aging, VAT compliance, advance-payments, annual-report status.
- [search](search.md) — global search: client resolution plus grouped record matching across eight domains.

## Cross-cutting & infra

- [audit](audit.md) — generic append-only entity audit trail (one read API, one log table).
- [users](users.md) — authentication, JWT lifecycle, user management, audit logs, password reset.
- [health](health.md) — `/health` + `/ready` liveness/readiness probes (no `/api/v1` prefix).
