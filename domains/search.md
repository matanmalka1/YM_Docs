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
Last verified against code: 2026-07-21.

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
- `SearchItem`: one record of any type, in the thin shape the feed renders — `result_type`, `id`,
  `title`, `detail`, `status`, `amount`, `occurred_on`, `href`. Client identity stays on the
  selected `SearchClientMatch`; repeating it on every item would add unused list payload.
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
  A binder that was handed over still resolves to its owner: where a binder is does not change
  whose it is, and the typed term identifies rather than filters. Soft-deleted binders never
  resolve anyone. The binder inside the term's `EXISTS` is aliased, because the statement also
  joins `Binder` whenever a binder filter is present.
- `matched_binder_numbers` explains a match that came from a binder number. It mirrors the term's
  own matching exactly, handed-over binders included — an explanation that omits the binder the
  user typed would be worse than none. It is empty for name or ID matches — there is nothing to
  explain then.
- The advanced filters (`client_record_id`, `id_number`, `client_status`, `entity_type`,
  `binder_number`, `binder_location_status`, `binder_capacity_status`) narrow the client list.
  Client filters and binder filters are combined with `AND`.
- The **binder filter** path is separate from the term: its join excludes soft-deleted and
  handed-over binders by default, and an explicit `binder_location_status=handed_over` searches
  handed-over binders instead. A term and a binder filter combine with `AND`, so filtering by
  location still narrows a client the term found through a handed-over binder.
- **Phase two — the item feed.** Items are always scoped to exactly one client, and only ever to
  a client that phase one returned. `client_record_id` requests a client; it does not select one.
  It narrows resolution like every other filter, so it selects that client only when resolution
  returns exactly it — a requested client the term or the advanced filters exclude selects
  nothing, and no group is loaded for it. Without a request, a term resolving to exactly one
  client auto-selects that client. While several clients match, every group is empty — a feed
  mixing two clients has no meaning.
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

- **2026-07-21 — the typed term identifies a client, it does not filter binders.** Searching a
  binder number used to skip handed-over binders, so a binder physically in the client's hands
  returned "no client found" while the same number found the client through the dedicated
  `binder_number` filter with `binder_location_status=handed_over`. The term now matches any
  non-deleted binder, and `matched_binder_numbers` mirrors it. The binder-filter path keeps its
  handed-over default — there it is a filter over binder state, which is what the user asked for.
  Fixed alongside it: the term's binder `EXISTS` auto-correlated to the outer `Binder` join, so
  any term combined with any binder filter raised at execution time and returned a 500.
- **2026-07-21 — a requested client cannot outrank client resolution.** `client_record_id` used to
  select the feed's client before the filtered client list was consulted, so a stale selection kept
  in the URL served that client's items while the same response reported no matching client. One
  screen then showed both "no client found" and the previous client's records. The rule is now
  single: the feed's client is the client resolution returned, for API clients and deep links
  alike. The frontend derives the selected client, its heading, the feed, and the empty state from
  that one resolved client, and clears the selection in the same URL write as any filter change
  that changes which clients resolve.
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
