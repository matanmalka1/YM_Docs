## Scope

This file owns only:

- The follow-up backlog for the unified Search page filter logic, from the 2026-06-28 read-through.

This file must not contain:

- Architecture/domain rules (those belong in `docs/domains/` + `docs/frontend/architecture.md` once a fix lands).
- Completed work (move to `docs/archive/` when done).

Source of truth: tracking only — not source of truth for current behavior.

# Search Filters — Follow-up (2026-06-28)

Backlog only. Captures why the Search page filters "don't play together". No code changed yet.

Risk legend: 🟡 contained fix · 🔴 design change (cross-entity query model).

## How the page works today (end to end)

- **URL is the single source of truth.** `useSearchParamFilters` reads/writes query params;
  `useSearchPage` rebuilds the `filters` object from the URL each render.
- **Two input zones.** Top free-text box (`search`, debounced) + collapsible advanced panel
  (`SearchFiltersBar`) with 8 keys: client picker, id_number, binder_number, client_status,
  entity_type, binder_location_status, binder_capacity_status, filename.
- **Gate.** The query is `enabled` only when `hasAnyFilter` (search OR any advanced key); otherwise
  the page shows the prompt `StateCard`.
- **One request, three payloads.** `GET /search` → `SearchService.search` returns
  `(results, total, documents)`. `results` feeds the main `DataTable`; `documents` feeds a separate
  `DocumentResultsSection` at the bottom.
- **Backend branch split** (`backend/app/search/services/search_service.py`):
  - `has_client_filter` = search | client_record_id | id_number | client_status | entity_type
  - `has_binder_filter` = search | client_record_id | binder_number | binder_location_status | binder_capacity_status
  - client-only → clean DB-level pagination.
  - anything binder-ish → "mixed" branch: build the full list in memory, then `paginate_sequence`.

## Items

### S1 — Mixed branch is a UNION, not an intersection · 🔴

`search_service.py` mixed branch runs two independent queries and concatenates them:

- client block filters by id/status/entity_type — **ignores capacity/location**.
- binder block filters by capacity/location/number — **ignores id_number/status/entity_type**; it only
  re-filters by `client_record_id` post-hoc.

Example: `id_number=1` + `client_status=active` + `binder_capacity_status=full` returns
(active clients whose id contains "1") **plus** (every full binder of every client). Capacity does not
narrow clients; status/id do not narrow binders. Additive sets read as noise.

Fix direction: a single `client → binder` joined query where every filter ANDs across the relation,
replacing the two-block concat. This is the root cause; S2/S4 fall out of it.

### S2 — Duplicate rows · 🔴 (resolved by S1)

The same client can surface twice — once as `result_type=client`, once as `result_type=binder`.
Distinct `getRowKey` (`${result_type}-${client_record_id}-${binder_id}`) means both render.

### S3 — `search` text silently leaks into `binder_number` · 🟡

`db_binder_number = binder_number or (search if not (client_record_id or id_number) else None)`.
Free-text is reused as a binder-number match, but only when no id/client is set — so the filter's
meaning changes depending on which other fields are filled. Drop the leak; match `search` against
binder number explicitly (or not at all) instead of conditionally.

### S4 — client_status / entity_type never applied to binder results · 🔴 (resolved by S1)

`status=active` still returns binders belonging to inactive clients, because the binder block never
filters by client attributes.

### S5 — Documents are a third parallel filter universe · 🟡

`documents` respond only to `search` / `filename` / `client_record_id`. id_number, status, entity_type,
capacity, location do nothing to the document section. The doc list below the table looks inconsistent
with the table above. Decide: either subject documents to the same client scope, or visually separate
them so the mismatch is expected.

### S6 — In-memory ceiling makes `total` / pagination lie on large sets · 🟡

Mixed branch caps at `_MIXED_SEARCH_BINDER_LIMIT=1000` / `_MIXED_SEARCH_CLIENT_LIMIT=500` and paginates
in memory; `total` counts only what survived the ceiling. Disappears once S1 moves filtering into a
single DB query with DB-level pagination.

## Root design flaw

Two entity types (clients, binders) plus documents are crammed into one endpoint with per-entity
filters OR'd together. There is no query model where filters intersect across the joined
client→binder relation; the frontend just renders whatever comes back. S1 is the real fix — S2/S4/S6
are symptoms of it; S3/S5 are independent contained cleanups.
