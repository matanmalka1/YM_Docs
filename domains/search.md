## Scope
This file owns only:
- Canonical current-state documentation for the search domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Search

Search answers "what is this thing?". One typed term is resolved two ways side by side: to
**clients** (client resolution) and to **records** of eight domains (record matching). The two
sections coexist in one response — there is no selection step and no per-client feed. Clicking
a client navigates to the client page, whose tabs are the dossier; clicking a match follows the
record's own deep link. Search is an orchestration layer over other domains and owns no
persisted tables. Specification: `docs/research/global-search-spec.md` (decisions D1–D8).
Last verified against code: 2026-07-21.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/search` | Resolve the term to clients (paginated) and return grouped record-match previews with exact per-type totals. |
| `GET` | `/api/v1/search/items` | One type's matches for the term, fully paginated — an expanded preview group. |

Both are defined in `backend/app/search/api/search_routes.py` under `prefix="/search"`.

**`GET /api/v1/search` params:** `search` (required; stripped; `min_length=1` after strip, so a
whitespace-only term is a 422; `max_length=200` as input hardening), `page`, `page_size`.
Pagination applies to the **clients** list; match previews are fixed-size. The seven former
params — `client_record_id`, `id_number`, `binder_number`, `client_status`, `entity_type`,
`binder_location_status`, `binder_capacity_status` — are deleted: gone from the signature and
from OpenAPI, not deprecated (ADR-0001/0003).

**`GET /api/v1/search/items` params:** `search` (same rules), `result_type` (required), `page`,
`page_size`. Returns `PaginatedResponse[SearchMatch]`. `result_type=client` remains a 422 via
the enum. The former `client_record_id` param is deleted — client-scoped listing is what the
client page's tabs do; search does not duplicate them.

## Model & fields

No `backend/app/search/models/` package and no search-owned persisted models; the repositories
read projections over models owned by the source domains.

Domain-owned response projections:

- `SearchClientMatch` (unchanged): one client the term resolved to — `id`,
  `office_client_number`, `name`, `id_number`, `status`, `matched_binder_numbers`, `href`.
- `SearchMatch`: one matched record of any type — `result_type`, `id`, `title`, `detail`,
  `status`, `amount`, `occurred_on`, `href`, `client_record_id`, `client_name`,
  `client_office_number`. A match row is meaningless without its client, so every row carries
  its client identity — unlike the deleted `SearchItem`, which was thinned because the dossier
  named its client once above the feed.
- `SearchMatchGroup`: up to `PREVIEW_LIMIT` (5) matches plus the exact total behind them.
- `SearchMatchGroups`: the eight groups — `binders`, `documents`, `vat_work_items`,
  `annual_reports`, `advance_payments`, `charges`, `tasks`, `notifications`.
- `SearchResponse`: `clients` (a `PaginatedResponse[SearchClientMatch]`) plus `matches`
  (a `SearchMatchGroups`).

`SearchItem`, `SearchItemGroup`, and `SearchItemGroups` are deleted — after the dossier feed
died (D2) they had no consumer.

## Enums / statuses

- `SearchMatchType` (`backend/app/search/schemas/search.py`; renamed from `SearchItemType`,
  same 8 members): `binder`, `document`, `vat_work_item`, `annual_report`, `advance_payment`,
  `charge`, `task`, `notification`. It deliberately excludes `client`: a client is a resolution
  result, not a record row, and `result_type=client` on `/search/items` is a 422.
- `LinkedEntity` (`backend/app/common/entity_links.py`, untouched): the `SearchMatchType`
  values plus `client`. It owns the frontend route each result links to.
- `SearchMatch.status` carries the source domain's own raw status value and is `null` for
  documents, which have no work status.
- The client/binder filter enums (`ClientStatus`, `EntityType`, `BinderLocationStatus`,
  `BinderCapacityStatus`) no longer appear anywhere in the search contract; they left with the
  deleted filter params.

## Domain rules & invariants

- Access is router-gated to `ADVISOR` and `SECRETARY` via `require_role(...)`. Search widens
  what can be found, never by whom.
- **Client resolution — unchanged behavior.** The term matches a client by official name, ID
  number, office client number, or the number of a binder it owns — one `OR` across the four
  identifiers (contains/`ilike`). A binder number resolves to its owning client because that is
  how the office locates a client from physical material. A handed-over binder still resolves
  its owner: where a binder is does not change whose it is. Soft-deleted binders never resolve
  anyone. Each client is returned exactly once no matter how many of its binders match, and the
  list is paginated with `page`/`page_size`.
- `matched_binder_numbers` explains a match that came from a binder number. It mirrors the
  term's own matching exactly, handed-over binders included. It is empty for name or ID
  matches — there is nothing to explain then.
- The internal `SearchFilters`/advanced-filter machinery was deleted together with the filter
  params: the resolution repository is term-only now.
- **Record matching — phase 1 is exact only.** Per-type matched columns:

  | Type | Phase-1 matched columns (exact) |
  |------|--------------------------------|
  | Charge | `id` (numeric term, user-visible) |
  | Binder | `binder_number` equality |
  | Document | `original_filename` equality |
  | VAT work item | `period` equality (normalized) |
  | Annual report | `tax_year` (numeric term, 4-digit year heuristic 1990–2100), `ita_reference` equality |
  | Advance payment | `period` equality (normalized) |
  | Task | `title` equality |
  | Notification | `recipient` equality (an email is typed whole) |

  Exclusions (D4): internal DB ids never participate — a column participates in identifier
  matching only if the UI displays it as the record's identifier. Amounts are not searchable
  (non-goal). `document_type` is not matched (stores English enum values; users type Hebrew
  labels). Notification trigger labels are not matched (a Python dict, not a column).
- **Term parsing.** One pure backend function (`backend/app/search/search_term_parser.py`),
  unit-testable without DB, classifies the trimmed term by capability: a **period-shaped** term
  (`03/2026`, `3/2026`, `2026-03`) is normalized to `2026-03` (D5) and activates the period
  branches *and* the text branches; a **bare integer** activates identifier branches only (D4),
  with integers bounded to int32; anything else activates text branches. Whitespace-only is not
  a search — 422 via `min_length` after strip. The frontend performs no term classification.
- **Repository.** `SearchMatchRepository` (replacing the deleted per-client
  `SearchItemRepository`) has a single branch builder per type. The grouped preview is one
  `UNION ALL` query across the eight branches with a rank column (every phase-1 row is tier 1),
  `row_number() OVER (PARTITION BY result_type ORDER BY ...) <= 5` for the previews, and
  `count(*) OVER (PARTITION BY result_type)` for the exact totals — previews and totals come
  from **one** query. Expansion (`/search/items`) runs the single relevant branch with real
  `LIMIT/OFFSET` + count. A resolved search request costs ≤ 4 queries total (the deleted
  dossier path cost 19).
- **Phase-1 usage measurement.** Every authenticated `GET /search` emits one
  `global_search_phase1_usage` structured log event after the existing queries complete. It
  records only the parsed term's length and classification (`period`, `integer`, or `text`), the
  resolved-client total, all eight per-type match totals, and whether the complete result was
  empty. The raw term is never logged because it can contain client names or identity numbers.
  The event reuses the response totals and adds no database query; its purpose is the phase-2
  gate in `docs/research/global-search-spec.md` §9.
- **Ordering** — identical in preview and expansion: rank, then the type's anchor date
  descending with NULLs last (via a portable boolean sort key), then `id` descending.
- **Phase-1 matching is exact and case-sensitive.** In particular, notification `recipient`
  email equality is case-sensitive — a product decision (2026-07-21), to be revisited with the
  phase-2 measurement.
- **Scoping — unchanged.** Soft-deleted records are excluded per domain (`deleted_at`), as are
  deleted and superseded permanent documents. Records belonging to a soft-deleted client record
  are never returned, on both endpoints. Notifications are unscoped beyond the client join.
- Every match carries `href`, built by `entity_route(...)`
  (`backend/app/common/entity_links.py`), which is the single owner of these URL strings.
  Advance payments, documents, and client-scoped annual reports link through their owning
  client's screen, which loads the single record and opens it regardless of that screen's own
  filters, year, or page.
- **Indexing (D7, staged).** The phase-1 Alembic migration adds only the missing btree indexes
  serving exact matching: `original_filename`, task `title`, notification `recipient`,
  `ita_reference`, `binder_number`. The `pg_trgm` + GIN migration is the entry gate to
  phase 2 — no `ILIKE` ships before the index that serves it exists.
- The work queue is **not** a match type. It is a derived urgency view over the same binders,
  charges, tasks, VAT items, annual reports, and advance payments; including it would return
  every urgent record twice.

## Error codes

No `SEARCH.*` domain error codes are raised inside `backend/app/search`. Search relies on shared
auth/validation errors (blank term and `result_type=client` are plain 422 validation errors)
and the global error-envelope rules in `docs/architecture/api-contracts.md`. Error-code
registry: `docs/backend/error-codes.md`.

## Known issues

- Matches stay grouped by type; there is no cross-type relevance merging (explicit spec
  non-goal). Ranking exists only inside a group, and in phase 1 every row is the same tier.
- Phase-1 matching is equality-only: prefix and contained matching (and any case-insensitive
  behavior) do not exist until phase 2 ships behind its gates.

## Resolved issues

- **F-032** (2026-06-05): Closed as stale. `app/search/README.md` is already a thin pointer to
  this canonical document.

## Decisions (preserved)

- **2026-07-21 — the dossier feed is deleted, not moved (D2).** Seven of its eight groups
  duplicated tabs that already exist on the client page; `/search` now navigates to
  `/clients/:id` and the tabs are the dossier. The `items` block, `SearchItem*` models, and the
  19-query per-client dossier path were deleted with it — closed debt, not a regression.
- **2026-07-21 — no auto-navigation, ever (D1).** Navigation happens only on click, whatever
  the match count. Auto-select was designed when selection was on-page state; with selection
  replaced by navigation, a debounce-triggered jump mid-typing is a trap.
- **2026-07-21 — the advanced filter params are deleted.** `/search` answers "what is this
  thing?" with identification only; client status/entity-type filtering belongs to `/clients`
  and binder location/capacity to `/binders`, where the resulting set is actionable. The seven
  params were removed from the signature and OpenAPI rather than deprecated (ADR-0001/0003).
- **2026-07-21 — identifier matching covers user-visible identifiers only (D4).** Charge id,
  binder number, office client number, ID number, period, tax year. Internal DB ids never
  participate: no screen shows them, so any row they return is noise.
- **2026-07-21 — period formats are normalized, not multiplied (D5).** The parser recognizes
  `03/2026`, `3/2026`, and `2026-03` and normalizes to the DB form `2026-03` before comparison.
  Users type what the screens show.
- **2026-07-21 — indexing is staged (D7).** Phase 1 (exact) ships btree-only. The `pg_trgm` +
  GIN migration is the entry gate to phase 2; no write cost is paid for capability that has not
  shipped.
- **2026-07-21 — phase-1 exact matching is case-sensitive**, including notification recipient
  email equality. Product decision; revisit with the phase-2 measurement of real terms.
- **2026-07-21 — the typed term identifies a client, it does not filter binders.** Searching a
  binder number used to skip handed-over binders, so a binder physically in the client's hands
  returned "no client found" while the same number found the client through the dedicated
  `binder_number` filter with `binder_location_status=handed_over`. The term now matches any
  non-deleted binder, and `matched_binder_numbers` mirrors it. The binder-filter path kept its
  handed-over default — there it was a filter over binder state, which is what the user asked
  for. (The filter path has since been deleted with the params; the term rule stands.)
  Fixed alongside it: the term's binder `EXISTS` auto-correlated to the outer `Binder` join, so
  any term combined with any binder filter raised at execution time and returned a 500.
- **2026-07-21 — a requested client cannot outrank client resolution.** `client_record_id` used to
  select the feed's client before the filtered client list was consulted, so a stale selection kept
  in the URL served that client's items while the same response reported no matching client. One
  screen then showed both "no client found" and the previous client's records. The rule became
  single: the feed's client is the client resolution returned. (The selection contract itself was
  later deleted with the dossier feed; the decision is preserved as the reason no selection
  machinery may return.)
- **2026-07-19 — search became client-centric.** The previous contract returned a joined
  client×binder table (`SearchResult` with `result_type: client|binder`), a separate document
  list, and a grouped `operational` block — three different shapes on one page, and one row per
  client/binder pair. It was replaced by client resolution plus one uniform item feed.
  `SearchResult`, `DocumentSearchResult`, the `filename` filter, and `DocumentSearchService`
  were removed rather than kept alongside the new contract.

## Future / planned

- **Phase 2 — prefix and contained matching** (`docs/research/global-search-spec.md` §9),
  gated on (a) the `pg_trgm` + GIN migration landing first and (b) measurement of phase-1 usage
  on real data (term distribution, zero-result rate, per-type noise). If exact matching covers
  the office's real queries, phase 2 may shrink or not ship.

## Historical notes

The dossier-feed contract (client resolution + one selected client's item feed, 2026-07-19 to
2026-07-21) was replaced by the matches contract specified in
`docs/research/global-search-spec.md`. The completed 2026-07-19 filter-refactor tracker is
preserved at `docs/archive/search-filters-refactor-2026-07-19.md`.
