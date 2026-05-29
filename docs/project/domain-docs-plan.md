## Scope

This file owns only:

- The plan, ownership split, and status tracker for authoring canonical per-domain docs.

This file must not contain:

- The domain docs themselves or authoring rules (those live in `docs/workflow/domain-docs-authoring.md`).

Source of truth: reference

# Domain Docs Plan

Tracks the migration to canonical `docs/docs/domains/<DOMAIN>.md`. Authoring rules + template: `docs/workflow/domain-docs-authoring.md`.

Two agents (Codex, Claude Code) work in parallel, each on a different domain. Each touches only its own files (see parallel rules in the authoring doc). Index/map registration is batched at the end.

Status legend: `todo` / `in-progress` / `done`.

## Notes

- `{MODULE}` = backend dir under `backend/app/`. Mapping is `-`↔`_` except `notifications` → module `notification` (singular).
- Pilot first: finish `advance-payments` and confirm the skeleton works before full parallel.
- Cross-cutting/infra domains (`audit`, `dashboard`, `search`, `reports`, `health`) get a minimal doc or may be skipped — decide per domain.

## Codex — tax chain + scheduling

| {DOMAIN}           | {MODULE}           | type         | status |
| ------------------ | ------------------ | ------------ | ------ |
| advance-payments   | advance_payments   | core (pilot) | done   |
| annual-reports     | annual_reports     | core         | todo   |
| vat-reports        | vat_reports        | core         | todo   |
| tax-calendar       | tax_calendar       | core         | todo   |
| charge             | charge             | core         | todo   |
| work-queue         | work_queue         | read         | todo   |
| notifications      | notification       | core         | todo   |
| signature-requests | signature_requests | core         | todo   |
| dashboard          | dashboard          | read         | todo   |
| search             | search             | cross-cut    | todo   |
| reports            | reports            | read         | todo   |
| health             | health             | infra        | todo   |

## Claude — client / operations

| {DOMAIN}            | {MODULE}            | type      | status |
| ------------------- | ------------------- | --------- | ------ |
| clients             | clients             | core      | todo   |
| businesses          | businesses          | core      | todo   |
| binders             | binders             | core      | todo   |
| tasks               | tasks               | core      | todo   |
| reminders           | reminders           | core      | todo   |
| notes               | notes               | core      | todo   |
| correspondence      | correspondence      | core      | todo   |
| permanent-documents | permanent_documents | core      | todo   |
| authority-contact   | authority_contact   | core      | todo   |
| timeline            | timeline            | read      | todo   |
| audit               | audit               | cross-cut | todo   |
| users               | users               | core      | todo   |

## Final batch (human / single owner, after parallel runs)

Collect every run's Registration block and update once:

- `docs/docs/domains/README.md` (create index)
- `docs/docs/project/documentation-map.md` (add Domains section)
