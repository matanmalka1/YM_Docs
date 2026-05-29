## Scope
This file owns only:
- Historical content from `backend/app/clients/README.md` prior to canonical doc authoring.

This file must not contain:
- Current-state documentation (see canonical doc).

Source of truth: historical

# Clients Module (Legacy — backend/app/clients/README.md)

> Archived from: `backend/app/clients/README.md` (last audited: 2026-04-21)
> Canonical domain doc: `docs/domains/clients.md`

---

Manages CRM clients as a two-layer identity model: `LegalEntity` (tax/legal identity, globally unique) and `ClientRecord` (office CRM anchor, one active record per legal entity). Business-level state (binders, VAT/report workflows, etc.) is handled under `app/businesses/` and related domains.

## Scope (original)

- CRUD for `ClientRecord` and underlying `LegalEntity`
- Optional `Person` + `PersonLegalEntityLink` for natural-person associations
- Search + pagination + status filtering for clients
- Soft delete + restore flow with audit columns
- Conflict detection by `id_number`
- Excel export/template/import endpoints
- Impact preview for client creation (projected obligations)

## Implementation Map (original)

- Models: `app/clients/models/legal_entity.py`, `app/clients/models/client_record.py`, `app/clients/models/person.py`, `app/clients/models/person_legal_entity_link.py`
- Schemas: `app/clients/schemas/client.py`, `app/clients/schemas/client_record_response.py`, `app/clients/schemas/impact.py`
- Repositories: `app/clients/repositories/client_record_repository.py`, `app/clients/repositories/client_record_read_repository.py`, `app/clients/repositories/legal_entity_repository.py`, `app/clients/repositories/person_repository.py`, `app/clients/repositories/client_repository.py`
- Services: `app/clients/services/client_service.py`, `app/clients/services/create_client_service.py`, `app/clients/services/client_query_service.py`, `app/clients/services/client_enrichment_service.py`, `app/clients/services/client_update_service.py`, `app/clients/services/client_lifecycle_service.py`, `app/clients/services/impact_preview_service.py`
- Guards: `app/clients/guards/client_record_guards.py`
- API: `app/clients/api/clients.py`, `app/clients/api/clients_excel.py`
- Enums: `app/clients/enums.py` (`ClientStatus`); entity/VAT/ID-type enums live in `app/common/enums.py`

## Tests (original)

- `tests/clients/api/test_clients.py`
- `tests/clients/api/test_clients_mutations_additional.py`
- `tests/clients/api/test_clients_excel.py`
- `tests/clients/api/test_onboarding_layer1.py`
- `tests/clients/service/test_client_service_mutations.py`
- `tests/clients/service/test_client_service_list_all_clients.py`
- `tests/clients/service/test_client_excel_service.py`
- `tests/clients/service/test_client_schema_and_model.py`
- `tests/clients/service/test_client_creation_service.py`
- `tests/clients/service/test_entity_type_change_guard.py`
- `tests/clients/repository/test_client_repository.py`
- `tests/clients/repository/test_client_record_repository.py`
