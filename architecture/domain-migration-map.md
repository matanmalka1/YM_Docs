# Domain Migration Map

Maps target domains (from `target-domains-v1.md`) to current codebase folders.

| Target Domain | Current Folder | Action | Priority | Notes |
|---|---|---|---|---|
| clients | clients/ | keep | P0 | |
| legal_entities | clients/models/ | extract | P1 | legal_entity.py, person.py, person_legal_entity_link.py |
| businesses | businesses/ | keep | P0 | |
| contacts | missing | create/extract | P1 | check if any code exists in clients/ |
| authority_contacts | authority_contact/ | rename | P2 | cosmetic only |
| notes | notes/ | keep | P0 | |
| documents | permanent_documents/ | create parent + move | P1 | permanent_documents/ becomes documents/permanent_documents/ |
| signature_requests | signature_requests/ | keep | P0 | |
| tasks | tasks/ | keep | P0 | |
| reminders | reminders/ | keep | P0 | |
| alerts | dashboard/ | extract | P2 | dashboard_attention_service.py |
| charges | charge/ | rename | P2 | cosmetic only |
| invoices | invoice/ | rename + add api layer | P1 | missing routers/ |
| advance_payments | advance_payments/ | keep | P0 | |
| vat | vat_reports/ | rename | P2 | cosmetic only |
| annual_reports | annual_reports/ | keep | P0 | |
| national_insurance | annual_reports/ | extract | P3 | ni_engine.py |
| tax_calendar | tax_calendar/ | keep | P0 | |
| binders | binders/ | keep | P0 | |
| communications | correspondence/ | rename | P2 | cosmetic only |
| notifications | notification/ | rename | P2 | cosmetic only |
| search | search/ | keep | P0 | |
| dashboard | dashboard/ | keep | P0 | after alerts extraction |
| work_queue | work_queue/ | keep | P0 | |
| actions | actions/ | restructure | P2 | flat → vertical slice |
| reports | reports/ | keep | P0 | |
| audit | audit/ | keep | P0 | |
| users | users/ | keep | P0 | |
| permissions | missing | defer | P3 | only when genuinely needed |
| imports | missing | defer | P3 | only when genuinely needed |
