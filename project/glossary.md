## Scope
This file owns only:
- Short disambiguation definitions for recurring terms, each with a pointer to its canonical owner doc.
- A Hebrew ↔ English term mapping.

This file must not contain:
- Domain behavior, rules, endpoints, or workflows (those live in the owner docs linked below).

Source of truth: reference

# Glossary

This glossary is a **disambiguation index, not a behavior source of truth**. Each entry is one short definition plus the doc that actually owns the term. When the glossary and an owner doc (or `backend/openapi.json`) disagree, the owner doc wins.

## 1. Identity & client terms

| Term | Short definition | Canonical owner |
|------|------------------|-----------------|
| ClientRecord | The CRM anchor row for a client engagement; the operational entity most domains attach to. | `docs/domains/clients.md` |
| client | Informal prose word for a ClientRecord. Not a distinct model. | `docs/domains/clients.md` |
| `client_record_id` | **Preferred/canonical operational anchor.** Use this as the filter/scoping field that points to `ClientRecord.id`. | `docs/architecture/api-contracts.md`, `docs/domains/clients.md` |
| `client_id` | Legacy/contextual name. In client route **path params** it means `ClientRecord.id`; in some endpoints (e.g. search) it is a real, intentional field. May appear in older prose/API contexts — **check the owning doc or `backend/openapi.json` before use, and do not globally rename real `client_id` fields.** | `docs/domains/clients.md`, `docs/domains/search.md` |
| LegalEntity | Stable legal/tax identity (unique by `id_number_type` + `id_number`); a ClientRecord points at one. | `docs/domains/legal-entities.md` |
| Person | A natural person in the identity graph. | `docs/domains/legal-entities.md` |
| PersonLegalEntityLink | Join row linking a Person to a LegalEntity (e.g. OWNER). | `docs/domains/legal-entities.md` |
| Business | A business/activity under a client's LegalEntity. | `docs/domains/businesses.md` |

## 2. Work item / operations terms

Distinguish **persisted domain records** from **computed/assembled operational views**.

| Term | Persisted or computed | Short definition | Canonical owner |
|------|-----------------------|------------------|-----------------|
| VAT work item | Persisted | A period-based VAT obligation record (`VatWorkItem`). | `docs/domains/vat.md` |
| charge | Persisted | An office billing charge with its own lifecycle. | `docs/domains/charges.md` |
| task | Persisted | A manual office task (standalone or source-linked). | `docs/domains/tasks.md` |
| reminder | Persisted | A scheduled reminder record (note: execution path has known issues). | `docs/domains/reminders.md` |
| action | Computed | Backend action metadata / cross-domain obligation orchestration helpers, not a stored row. | `docs/domains/actions.md` |
| work-queue item | Computed | An entry in the assembled open-work feed, derived at read time from tax/billing/binder sources plus tasks; not its own table. | `docs/domains/work-queue.md` |

## 3. History / audit terms

| Term | Short definition | Canonical owner |
|------|------------------|-----------------|
| AuditTrail | Compliance/accountability event recording for audited entity changes. | `docs/domains/audit.md` |
| Timeline | Operational, client-facing chronological activity feed; separate from AuditTrail. | `docs/domains/timeline.md` |
| status history | Domain-local status-change log (e.g. `AnnualReportStatusHistory`); scoped to its own domain, not a shared concept. | owning domain doc (e.g. `docs/domains/annual-reports.md`) |
| report history | **Retired terminology.** Do not revive unless a current domain doc explicitly reintroduces it. | n/a (historical; see `docs/archive/`) |

## 4. Hebrew ↔ English mapping

Common mappings already used across the docs and error-code namespaces. For behavior, follow the owner doc, not this table.

| Hebrew | English / model |
|--------|-----------------|
| לקוח | ClientRecord |
| ישות משפטית | LegalEntity |
| אדם | Person |
| עסק | Business |
| מע״מ | VAT |
| מקדמות | advance payments |
| דוח שנתי | annual report |
| חיוב | charge |
| חשבונית | invoice |
| קלסר | binder |
| תזכורת | reminder |
| התראה | notification / alert — check the owning domain (`docs/domains/notifications.md` vs `docs/domains/alerts.md`) |
| משימה | task |
| תור עבודה | work queue |
