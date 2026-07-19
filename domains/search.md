## Scope
This file owns only:
- Canonical current-state documentation for the search domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Search

Search is a two-phase, client-centric locator. Phase one resolves what the user typed to a
client record. Phase two returns that client's records from every domain in one shared item
shape, each carrying a deep link to the record itself. It is an orchestration layer over other
domains and owns no persisted tables.
Last verified against code: 2026-07-19.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/search` | Resolve the term/filters to clients, and preview the selected client's items per type. |
| `GET` | `/api/v1/search/items` | One client's items of a single type, fully paginated — an expanded preview group. |

Both are defined in `backend/app/search/api/search_routes.py` under `prefix="/search"`.

## Model & fields

No `backend/app/search/models/` package and no search-owned persisted models; the repositories
read projections over models owned by the source domains.

Domain-owned response projections:

- `SearchClientMatch`: one client the term resolved to — `id`, `office_client_number`, `name`,
  `id_number`, `status`, `matched_binder_numbers`, `href`.
- `SearchItem`: one record of any type, in the shape every type shares — `result_type`, `id`,
  `client_record_id`, `office_client_number`, `client_name`, `title`, `detail`, `status`,
  `amount`, `href`.
- `SearchItemGroup`: up to `PREVIEW_LIMIT` (5) items plus the exact total behind them.
- `SearchItemGroups`: the eight groups — `binders`, `documents`, `vat_work_items`,
  `annual_reports`, `advance_payments`, `charges`, `tasks`, `notifications`.
- `SearchResponse`: `clients` (a `PaginatedResponse[SearchClientMatch]`) plus `items`.

## Enums / statuses

- `SearchItemType` (`backend/app/search/schemas/search.py`): `binder`, `document`,
  `vat_work_item`, `annual_report`, `advance_payment`, `charge`, `task`, `notification`. It
  deliberately excludes `client`: the client is the feed's subject, not a row in it, and
  `result_type=client` on `/search/items` is a 422.
- `LinkedEntity` (`backend/app/common/entity_links.py`): the `SearchItemType` values plus
  `client`. It owns the frontend route each result links to.
- Client-resolution filters accept `ClientStatus` (`active`, `frozen`, `closed`), `EntityType`
  (`osek_patur`, `osek_murshe`, `company_ltd`, `employee`), `BinderLocationStatus` (`in_office`,
  `ready_for_handover`, `handed_over`), and `BinderCapacityStatus` (`open`, `full`).
- `SearchItem.status` carries the source domain's own raw status value and is `null` for
  documents, which have no work status.

## Domain rules & invariants

- Access is router-gated to `ADVISOR` and `SECRETARY` via `require_role(...)`.
- **Phase one — client resolution.** `search` matches a client by official name, ID number,
  office client number, or the number of a binder it owns. A binder number resolves to its
  owning client because that is how the office locates a client from physical material. Each
  client is returned exactly once no matter how many of its binders match, and the list is
  paginated with `page`/`page_size`.
- `matched_binder_numbers` explains a match that came from a binder number. It is empty for
  name or ID matches — there is nothing to explain then.
- The advanced filters (`client_record_id`, `id_number`, `client_status`, `entity_type`,
  `binder_number`, `binder_location_status`, `binder_capacity_status`) narrow the client list.
  Client filters and binder filters are combined with `AND`.
- Binder joins exclude soft-deleted and handed-over binders by default; an explicit
  `binder_location_status=handed_over` searches handed-over binders instead.
- **Phase two — the item feed.** Items are always scoped to exactly one client. An explicit
  `client_record_id` selects it; otherwise a term resolving to exactly one client auto-selects
  that client. While several clients match, every group is empty — a feed mixing two clients
  has no meaning.
- The typed term does **not** filter items. Once a client is selected the feed shows that
  client's records in full, and narrowing happens by type. "All of this client's data" is the
  contract.
- Each group returns at most 5 preview items and an exact total. `/search/items` returns one
  type in full with real pagination.
- Every item carries `href`, built by `entity_route(...)`, which is the single owner of these
  URL strings. Advance payments, documents, and client-scoped annual reports link through their
  owning client's screen, which loads the single record and opens it regardless of that
  screen's own filters, year, or page.
- Soft-deleted records are excluded per domain (`deleted_at`), as are deleted and superseded
  permanent documents. Items belonging to a soft-deleted client record are never returned.
- The work queue is **not** an item type. It is a derived urgency view over the same binders,
  charges, tasks, VAT items, annual reports, and advance payments; including it would return
  every urgent record twice.

## Error codes

No `SEARCH.*` domain error codes are raised inside `backend/app/search`. Search relies on shared
auth/validation errors and the global error-envelope rules in
`docs/architecture/api-contracts.md`. Error-code registry: `docs/backend/error-codes.md`.

## Known issues

- Item ordering is per type (most recent period or update first). There is no cross-type
  relevance ranking, because the feed is grouped by type rather than merged.

## Resolved issues

- **F-032** (2026-06-05): Closed as stale. `app/search/README.md` is already a thin pointer to
  this canonical document.

## Decisions (preserved)

- **2026-07-19 — search became client-centric.** The previous contract returned a joined
  client×binder table (`SearchResult` with `result_type: client|binder`), a separate document
  list, and a grouped `operational` block — three different shapes on one page, and one row per
  client/binder pair. It was replaced by client resolution plus one uniform item feed.
  `SearchResult`, `DocumentSearchResult`, the `filename` filter, and `DocumentSearchService`
  were removed rather than kept alongside the new contract.

## Future / planned

No planned search-domain behavior is documented here.

## Historical notes

The completed 2026-07-19 filter-refactor tracker is preserved at
`docs/archive/search-filters-refactor-2026-07-19.md`.
