## Scope

Archived completion record for the unified Search filter refactor originally tracked in `docs/search-filters-followup.md`.

Source of truth: historical only. Current behavior is owned by `docs/domains/search.md`.

# Search Filters Refactor — completed 2026-07-19

The refactor closed the six issues from the 2026-06-28 follow-up:

- Replaced the client-result plus binder-result in-memory UNION with one joined, DB-paginated projection.
- Client and binder filters now intersect with `AND`.
- Removed duplicate client + binder representations of the same relation.
- Broad `search` explicitly matches binder number but is no longer conditionally reused as the dedicated `binder_number` filter.
- Applied the advanced client/binder scope to document and operational result sections.
- Removed the 500-client/1000-binder in-memory ceilings; `total` is counted from the filtered DB statement.

Regression coverage locks combined status/capacity filtering, free-text client matching in the presence of an unrelated binder number, exact pagination, document scoping, operational scoping, and the single-row client/binder representation.
