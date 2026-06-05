## Scope
This file owns only:
- Canonical current-state documentation for the search domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Search

The search domain exposes one role-gated unified endpoint that composes client, binder, and permanent-document lookups into a single response. It is an orchestration layer over other domains' repositories and does not own persisted tables of its own.
Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/search` | Unified search across client records, active binders, and matching permanent documents with optional filters and pagination. |

The module router is defined with `prefix="/search"` at `backend/app/search/api/search.py:11-18`, and the effective path exists in `backend/openapi.json:7484-7673`.

## Model & fields

This domain has no `backend/app/search/models/` package and no search-owned persisted SQLAlchemy models; `backend/app/search` contains only `api/`, `schemas/`, and `services/` directories (`find backend/app/search -maxdepth 2 -type d`).

The domain-owned response projections are:

- `SearchResult` (`backend/app/search/schemas/search.py:4-14`): `result_type` (non-null `str`), `client_id` (non-null `int`), `office_client_number` (`int | None`), `client_name` (non-null `str`), `id_number` (`str | None`), `client_status` (`str | None`), `binder_id` (`int | None`), `binder_number` (`str | None`).
- `DocumentSearchResult` (`backend/app/search/schemas/search.py:17-28`): `id` (non-null `int`), `client_record_id` (non-null `int`), `office_client_number` (`int | None`), `client_name` (non-null `str`), `business_id` (`int | None`), `business_name` (`str | None`), `document_type` (non-null `str`), `original_filename` (`str | None`), `tax_year` (`int | None`).
- `SearchResponse` (`backend/app/search/schemas/search.py:31-38`): `results` (non-null list), `documents` (non-null list with default empty list), `page` (non-null `int`), `page_size` (non-null `int`), `total` (non-null `int`).

## Enums / statuses

Search accepts these real enum values from code:

- `ClientStatus`: `active`, `frozen`, `closed` (`backend/app/clients/enums.py:4-7`).
- `EntityType`: `osek_patur`, `osek_murshe`, `company_ltd`, `employee` (`backend/app/common/enums.py:20-33`).
- `BinderLocationStatus`: `in_office`, `ready_for_handover`, `handed_over` (`backend/app/binders/models/binder.py:20-24`).
- `BinderCapacityStatus`: `open`, `full` (`backend/app/binders/models/binder.py:26-28`).

Returned `SearchResult.result_type` values are `client` and `binder`, as emitted by `SearchService.search()` (`backend/app/search/services/search_service.py:75-90,118-128,157-168`) and described by the schema comment at `backend/app/search/schemas/search.py:7`.

## Domain rules & invariants

- Access is router-gated to `ADVISOR` and `SECRETARY` via `require_role(...)` (`backend/app/search/api/search.py:11-15`).
- The endpoint accepts optional filters `query`, `client_name`, `id_number`, `binder_number`, `client_status`, `entity_type`, `binder_location_status`, `binder_capacity_status`, `filename`, plus paginated `page>=1` and `1<=page_size<=100` (`backend/app/search/api/search.py:18-32`; `backend/openapi.json:7497-7664`).
- Document matches are produced whenever either `query` or `filename` is present; otherwise `documents` is an empty list (`backend/app/search/services/search_service.py:50-53`).
- If there is at least one client-side filter and no binder-side filter, search uses `ClientRecordRepository.search(...)` with DB-level pagination and returns only `client` rows in `results` (`backend/app/search/services/search_service.py:55-95`).
- Any binder-side search path is assembled in memory, then paginated after result construction; the service hard-limits mixed-mode client fetches to `500` and binder fetches to `1000` (`backend/app/search/services/search_service.py:13-17,97-173`).
- In binder mode, `query` is reused as a binder-number filter only when `binder_number` is absent and neither `client_name` nor `id_number` is provided (`backend/app/search/services/search_service.py:131-140`).
- Active-binder enrichment for client rows comes from `map_active_by_clients(...)`, which returns only binders that are `in_office`, `open`, and not soft-deleted (`backend/app/search/services/search_service.py:71-72`; `backend/app/binders/repositories/binder_repository.py:501-525`).
- Binder result rows come from `list_active(...)`, which excludes handed-over binders unless `binder_location_status == handed_over`, filters by optional location/capacity/binder number, and paginates before returning (`backend/app/search/services/search_service.py:131-141`; `backend/app/binders/repositories/binder_repository.py:156-205`).
- Document search is capped at `50` results, searches filename-only when `filename` is present without `query`, otherwise searches `original_filename` or `document_type`, and returns only non-deleted, non-superseded documents (`backend/app/search/services/document_search_service.py:10,21-49`; `backend/app/permanent_documents/repositories/permanent_document_repository.py:158-185`).
- Document search enriches each result with `office_client_number` and `client_name` from `get_full_records_bulk(...)`, which itself filters to non-deleted client records (`backend/app/search/services/document_search_service.py:28-47`; `backend/app/clients/repositories/client_record_read_repository.py:117-126`).

## Error codes

No `SEARCH.*` domain error codes are raised inside `backend/app/search`; `rg -n 'SEARCH\\.|search\\.' backend/app/search` returns only imports and symbol names. Search relies on shared auth/validation errors and the global error-envelope rules in `docs/architecture/api-contracts.md`. Error-code registry: `docs/architecture/error-codes.md`.

## Known issues

No open known issues.

## Resolved issues

- **F-032** (2026-06-05): Closed as stale. `app/search/README.md` is already a thin pointer to this canonical document — no action required.

## Decisions (preserved)

No still-valid search-specific decisions were found in `backend/docs/domain_decisions_v3.md` or any search-specific `backend/docs/**` legacy file. Current enforced behavior is documented in the sections above.

## Future / planned

No search-specific future/planned behavior could be verified from `backend/docs/domain_decisions_v3.md` or search-specific legacy docs. Nothing is documented here as current unless it is implemented in code.

## Historical notes

No search-specific `backend/docs/**` legacy file was found, so no archive file was created for this domain.
