# Domain Migration Map

Maps target domains (from `target-domains-v1.md`) to current codebase folders.

| Target Domain | Current Folder | Action | Priority | Notes |
|---|---|---|---|---|
| clients | clients/ | keep | P0 | |
| legal_entities | legal_entities/ | keep | P1 done | Phase 1 extracted models and repositories from clients/ |
| businesses | businesses/ | keep | P0 | |
| contacts | contacts/ | keep | P1 done | Phase 6 scaffold only |
| authority_contacts | authority_contacts/ | keep | P2 done | Phase 7.2 renamed from authority_contact/ |
| notes | notes/ | keep | P0 | |
| documents | documents/permanent_documents/ | keep | P1 done | Phase 5 created parent and moved permanent_documents/ |
| signature_requests | signature_requests/ | keep | P0 | |
| tasks | tasks/ | keep | P0 | |
| reminders | reminders/ | keep | P0 | |
| alerts | alerts/ | keep | P2 done | Phase 3 extracted alert_service.py from dashboard/ |
| charges | charges/ | keep | P2 done | Phase 7.4 renamed from charge/ |
| invoices | invoices/ | keep | P2 done | Phase 7.5 renamed from invoice/ |
| advance_payments | advance_payments/ | keep | P0 | |
| vat | vat/ | keep | P2 done | Phase 7.6 renamed from vat_reports/ |
| annual_reports | annual_reports/ | keep | P0 | |
| national_insurance | annual_reports/ | extract | P3 | ni_engine.py |
| tax_calendar | tax_calendar/ | keep | P0 | |
| binders | binders/ | keep | P0 | |
| communications | communications/ | keep | P2 done | Phase 7.1 renamed from correspondence/ |
| notifications | notifications/ | keep | P2 done | Phase 7.3 renamed from notification/ |
| search | search/ | keep | P0 | |
| dashboard | dashboard/ | keep | P0 | after alerts extraction |
| work_queue | work_queue/ | keep | P0 | |
| actions | actions/ | keep | P2 done | Phase 4 moved flat modules under services/ |
| reports | reports/ | keep | P0 | |
| audit | audit/ | keep | P0 | |
| users | users/ | keep | P0 | |
| permissions | missing | defer | P3 | only when genuinely needed |
| imports | missing | defer | P3 | only when genuinely needed |
