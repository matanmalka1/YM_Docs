## Scope
This file owns only:
- Canonical current-state documentation for the search domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Search

The search domain exposes one role-gated unified endpoint that composes client/binder projections, permanent documents, and operational items into one response. It is an orchestration layer over other domains and does not own persisted tables of its own.
Last verified against code + backend/openapi.json: 2026-07-19.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/search` | Unified search across client/binder relations, documents, tasks, VAT work items, annual reports, charges, and advance payments. |

The module router is defined with `prefix="/search"` in `backend/app/search/api/search_routes.py`, and the effective path is published in `backend/openapi.json`.

## Model & fields

This domain has no `backend/app/search/models/` package and no search-owned persisted SQLAlchemy models; its repositories build read-only projections over models owned by the source domains.

The domain-owned response projections are:

- `SearchResult`: one DB-projected client/binder relation. `result_type` is `client` when there is no matching binder and `binder` when binder identity is present; the row always carries the owning client's identity and may carry binder identity.
- `DocumentSearchResult` (`backend/app/search/schemas/search.py:17-28`): `id` (non-null `int`), `client_record_id` (non-null `int`), `office_client_number` (`int | None`), `client_name` (non-null `str`), `business_id` (`int | None`), `business_name` (`str | None`), `document_type` (non-null `str`), `original_filename` (`str | None`), `tax_year` (`int | None`).
- `OperationalSearchItem`: a thin cross-domain result carrying its type, entity id, owning client identity, display title/detail, raw status, optional amount, and frontend deep link.
- `OperationalSearchGroup`: up to 5 preview items plus the full matching `total` for one domain.
- `OperationalSearchResults`: grouped `tasks`, `vat_work_items`, `annual_reports`, `charges`, and `advance_payments`.
- `SearchResponse`: the existing `results` and `documents`, the grouped `operational` result block, and primary-result pagination metadata.

## Enums / statuses

Search accepts these real enum values from code:

- `ClientStatus`: `active`, `frozen`, `closed` (`backend/app/clients/client_enums.py`).
- `EntityType`: `osek_patur`, `osek_murshe`, `company_ltd`, `employee` (`backend/app/common/enums.py:20-33`).
- `BinderLocationStatus`: `in_office`, `ready_for_handover`, `handed_over` (`backend/app/binders/models/binder.py:20-24`).
- `BinderCapacityStatus`: `open`, `full` (`backend/app/binders/models/binder.py:26-28`).

Returned `SearchResult.result_type` values are `client` and `binder`, as projected by `SearchResultRepository.search_primary(...)` and constrained by the schema literal.

## Domain rules & invariants

- Access is router-gated to `ADVISOR` and `SECRETARY` via `require_role(...)` in `backend/app/search/api/search_routes.py`.
- `SearchResult.client_name` always identifies the owning client, including for binder results. A business name must not replace it because a binder belongs to the client record and may contain material spanning all of that client's businesses.
- The endpoint accepts optional filters `search`, `client_record_id`, `id_number`, `binder_number`, `client_status`, `entity_type`, `binder_location_status`, `binder_capacity_status`, `filename`, plus paginated `page>=1` and `1<=page_size<=100`.
- `search` is the broad/free-text parameter. In primary results it explicitly matches client name, ID number, office client number, or binder number. It is never silently reassigned to the dedicated `binder_number` filter.
- Primary search is one joined, DB-paginated projection owned by `SearchResultRepository`. Client filters and binder filters are combined with `AND`; the exact `total` comes from the same filtered statement. There is no in-memory concatenation or safety ceiling.
- Each client/binder relation is emitted once. A client with a matching binder is not also emitted as a second client-only row. Multiple rows for one client are possible only when distinct binders independently match the filters.
- `client_record_id` scopes results to an exact client record. For `search + client_record_id`, permanent-document matches cannot return documents belonging to other clients.
- Document matches are produced whenever either `search` or `filename` is present; otherwise `documents` is an empty list.
- A non-empty `search` value also searches operational items across task title/description, VAT period, annual-report year/reference, charge period/description, and advance-payment period/notes. All groups additionally match entity id, client name, office client number, client ID number, and an active binder number belonging to the client.
- The advanced client/binder filters define a shared client scope for documents and operational groups. Thus client status/entity type and binder number/location/capacity constrain those secondary result sections as well as the primary table.
- `client_record_id`, when supplied, scopes every result section to that exact client. Selecting a client without free text returns the latest operational previews for that client. Each operational group returns at most 5 preview rows and an exact count; it does not flatten unlike entity types into the primary paginated `results` list.
- Operational result links identify the concrete entity. Advance-payment links carry `advance_payment_id`; the client advance-payments screen loads that record through its client-owned detail endpoint and opens its drawer even when the record is outside the currently displayed year, filters, or page.
- Primary binder joins exclude soft-deleted and handed-over binders by default. An explicit `binder_location_status=handed_over` searches handed-over binders instead.
- Document search is capped at `50`, returns only non-deleted/non-superseded documents, applies `search` to filename or document type, applies `filename` to filename, and combines both with `AND` when both are present.
- Document search enriches each result with `office_client_number` and `client_name` from `get_full_records_bulk(...)`, which itself filters to non-deleted client records (`backend/app/search/services/search_document_search_service.py`; `backend/app/clients/repositories/client_record_read_repository.py`).

## Error codes

No `SEARCH.*` domain error codes are raised inside `backend/app/search`; `rg -n 'SEARCH\\.|search\\.' backend/app/search` returns only imports and symbol names. Search relies on shared auth/validation errors and the global error-envelope rules in `docs/architecture/api-contracts.md`. Error-code registry: `docs/backend/error-codes.md`.

## Known issues

- Operational groups are intentionally previews: each returns five rows and an exact total. There is no group-specific pagination inside the Search page.

## Resolved issues

- **F-032** (2026-06-05): Closed as stale. `app/search/README.md` is already a thin pointer to this canonical document — no action required.

## Decisions (preserved)

No still-valid search-specific decisions were found in `backend/docs/domain_decisions_v3.md` or any search-specific `backend/docs/**` legacy file. Current enforced behavior is documented in the sections above.

## Future / planned

No planned search-domain behavior is documented here.

## Historical notes

The completed 2026-07-19 filter-refactor tracker is preserved at `docs/archive/search-filters-refactor-2026-07-19.md`.
