> **Specification. Approved for implementation planning; not yet implemented.**
> Current search behavior lives in `docs/domains/search.md` ★. That document changes with the
> code, in the same task that implements each phase. Until then, nothing here describes the
> running system.
>
> This document **replaces** `global-search-proposal.md` (the 2026-07-21 draft). The draft
> predated the page-division decision and the death of the dossier feed; where they disagree,
> this document wins.

# Global Search — Specification

**Written:** 2026-07-21
**Decision owner:** product (Matan)
**Status:** decided — all open questions below are closed
**Scope:** `/search` becomes "find anything"; the dossier feed dies; the client page's tabs are
the dossier.

---

## 1. Decisions log

Every open question raised during review, with its ruling. These are product decisions, not
implementation details — do not re-litigate them during implementation.

| # | Question | Ruling |
|---|----------|--------|
| D1 | Auto-navigation when the term resolves to one client | **Never.** Navigation happens only on click. Auto-select was designed when selection was on-page state; with selection replaced by navigation, a debounce-triggered jump mid-typing is a trap. |
| D2 | Where does the dossier live | **The dossier feed dies.** 7 of its 8 groups duplicate tabs that already exist on the client page (`CLIENT_DETAILS_TABS` has documents, charges, vat, advance-payments, annual-reports, tasks, notifications, timeline…). `/search` navigates to `/clients/:id`; the tabs are the dossier. No "Overview" feed tab is built. |
| D3 | Dossier query cost | **Moot** — follows from D2. The 19-query dossier path is deleted, not optimized. |
| D4 | Numeric-term identifier scope | **User-visible identifiers only**: charge id, binder number, office client number, ID number (ת.ז/ח.פ), period, tax year. Internal DB ids (task/notification/document/VAT-item ids) never participate — no screen shows them, so any row they return is noise. Rule: a column participates in identifier matching only if the UI displays it as the record's identifier. |
| D5 | Period format | **Normalize.** The parser recognizes `03/2026`, `3/2026`, and `2026-03` and normalizes to the DB form `2026-03` before comparison. Users type what the screens show. |
| D6 | Notifications in match scope | **Included**, matched on `subject_snapshot` and `recipient` — the only real searchable columns. Keeps all eight types behaving the same. |
| D7 | Index phasing | **Staged.** Phase 1 (exact match) adds only missing btree indexes. The `pg_trgm` + GIN migration is the *entry gate* to phase 2 — no `ILIKE` ships before the index that serves it exists. No write cost is paid for capability that has not shipped. |
| D8 | Binders gap on the client page | **New binders tab.** Every other domain already follows the pattern global-list + client-tab, and `GET /api/v1/clients/{id}/binders` already exists — frontend-only work. |

### Non-goals (explicit)

- **Combined multi-token queries** (`משה 06/2026`). The term is compared as one unit. The
  "record of a known client" scenario is two clicks: term → client → tab. Candidate for a
  future phase only if real usage demands it.
- **Amount search** (`1,200`). Amounts are `Decimal`, not identifiers; dozens of records share
  one amount; and every numeric term would ambiguously mean both id and amount. If a real need
  appears, it is a separate feature with a dedicated amount input, not the text box.
- **Internal id lookup** (D4).
- **Cross-type relevance merging.** Matches stay grouped by type; there is no unified ranked
  list across types.
- **Notification trigger-label search.** `TRIGGER_LABELS` is a Python dict, not a column;
  it cannot appear in a `WHERE` clause. Not worked around.
- **Identifier invention.** Periodic records (VAT, advance payments, annual reports) are
  identified in the real world by (client, period/tax-year) — matching how the ITA refers to
  them. No serial number is surfaced for them. If the office ever needs a one-word reference,
  the fix is to display a number in the UI, which then enters search under D4 — never to
  expose an internal id.

---

## 2. The division of labor between pages

Decided before this spec; the spec must conform to it:

| Page | Question it answers | Free text | Filters |
|---|---|---|---|
| `/search` | *What is this thing?* | **Identification** — clients and records | **None** |
| `/clients` | *Which clients?* | Narrows the list | status, entity type, accountant |
| `/clients/:id` | *What about this client?* | None | Per tab |
| Domain lists | *What are we working on?* | None | Produce an actionable set |

Consequences, all decided:

1. **`/search` loses client mode.** Clicking a client navigates to `/clients/:id`. The page
   becomes results-only. The `client_record_id` selection machinery on the search page —
   including parts of MAT-63/MAT-64 — is deleted. This is **closed debt, not a regression**:
   the selection contract those fixes hardened no longer exists to be violated.
2. **The dossier feed dies** (D2). It solved "show everything of the client" on the wrong
   page; the client page's tabs already solve it on the right one.
3. **All four advanced filters are deleted from `/search`.** Client status and entity type
   belong to `/clients`; binder location/capacity belong to `/binders`, where the resulting
   set is actionable.

---

## 3. What the typed term matches

### 3.1 Client matching — unchanged

Official name, ID number (ת.ז/ח.פ), office client number, binder number — one `OR`, exactly
as today, including the 2026-07-21 rulings (handed-over binders resolve their owner;
`matched_binder_numbers` mirrors the term's matching; soft-deleted binders never resolve).

### 3.2 Record matching — new, per type

Only columns `SearchItemRepository` already reads. No new joins.

| Type | Phase 1 (exact) | Phase 2 (prefix / contained) | Excluded, and why |
|---|---|---|---|
| Charge | `id` (numeric term, user-visible) | `description` contained | `amount` (non-goal) |
| Binder | `binder_number` equality | `binder_number` prefix; `notes` contained | — |
| Document | `original_filename` equality | `original_filename` prefix/contained | `document_type` — column stores English enum values (`bank_approval`); users type Hebrew labels. Matching it would be dead code. |
| VAT work item | `period` equality (normalized, D5) | — (period is the only searchable column) | internal `id` (D4) |
| Annual report | `tax_year` (numeric term, 4-digit year heuristic), `ita_reference` equality | `ita_reference` prefix | internal `id` (D4) |
| Advance payment | `period` equality (normalized, D5) | `notes` contained | internal `id` (D4) |
| Task | `title` equality | `title`/`description` contained | internal `id` (D4) |
| Notification | `recipient` equality (an email is typed whole) | `subject_snapshot` contained | trigger label (not a column); internal `id` (D4) |

### 3.3 Term parsing

One pure backend function (unit-testable without DB) classifies the trimmed term:

- **Period-shaped** (`MM/YYYY`, `M/YYYY`, `YYYY-MM`) → normalized to `YYYY-MM`; activates the
  period branches *and* the plain-text branches (a filename can legitimately contain
  `2026-03`; in phase 1 with equality-only this is theoretical, in phase 2 it matters).
- **Bare integer** → identifier branches per D4: charge id, office client number (client
  side), ID number (client side), tax year if it parses as a plausible year (1990–2100).
  In phase 2 also prefix over textual identifiers (binder number).
- **Anything else** → text branches (filename, title, notes, subject, recipient…).
- **Whitespace-only** → not a search. Frontend never sends it; API rejects it (422 via
  `min_length` after strip).

Frontend performs no term classification — it sends the trimmed term as-is. The parser lives
in one place.

---

## 4. Ranking

Three tiers, ordered:

1. **Exact identifier** — equality on the D4 identifier set, normalized period, full filename,
   exact title/recipient.
2. **Prefix** — textual identifiers starting with the term (phase 2).
3. **Contained** — free text containing the term (phase 2).

Within a tier: most recent first, using each type's existing anchor date (the same
`occurred_on` column the type is ordered by today). Phase 1 ships tier 1 only, so the rank
column exists in the SQL from day one but every phase-1 row is tier 1.

The rank tier is **not** exposed in the API response — ordering is the server's job, and the
prototype's "show ranking" toggle was a demo aid, not a feature.

---

## 5. API contract

### 5.1 `GET /api/v1/search` — becomes matches-only

**Params:** `search` (required, `min_length=1` after strip), `page`, `page_size`
(pagination applies to the `clients` list, as today).

**Deleted params** — the page was their only consumer; keeping them is legacy (ADR-0001/0003):
`client_record_id`, `id_number`, `binder_number`, `client_status`, `entity_type`,
`binder_location_status`, `binder_capacity_status`.

**Response:**

```
SearchResponse
├── clients: PaginatedResponse[SearchClientMatch]   # unchanged shape
└── matches: SearchMatchGroups                      # NEW — replaces `items`
```

`SearchItemGroups`/`items` (the dossier block) is **deleted**, not deprecated — after D2 it
has no consumer.

### 5.2 New response models

`SearchItem` was deliberately thinned on 2026-07-21 (client identity removed) because the
dossier named its client once. A match row is meaningless without its client, so matches get
their **own model** rather than re-fattening a shape that no longer exists:

```
SearchMatch
├── result_type: SearchMatchType      # renamed from SearchItemType, same 8 members
├── id: int
├── title: str                        # the type's key: binder number, filename, period…
├── detail: str | null
├── status: str | null                # null for documents, as today
├── amount: ApiDecimal | null
├── occurred_on: date | null
├── href: str                         # entity_route(...), unchanged ownership
├── client_record_id: int             # NEW — match rows carry their client
├── client_name: str                  # NEW
└── client_office_number: int | null  # NEW

SearchMatchGroup  { items: list[SearchMatch], total: int }   # ≤ PREVIEW_LIMIT (5) rows
SearchMatchGroups { binders, documents, vat_work_items, annual_reports,
                    advance_payments, charges, tasks, notifications }
```

`SearchItem`, `SearchItemGroup`, `SearchItemGroups` are deleted. `SearchClientMatch` is
unchanged. `SearchMatchType` keeps excluding `client` (a client is a resolution result, not a
record row); `LinkedEntity` is untouched.

### 5.3 `GET /api/v1/search/items` — retargeted to match expansion

Was: one *client's* items of one type. Becomes: one *type's* matches for the term, in full.

**Params:** `search` (required), `result_type` (required), `page`, `page_size`.
**Deleted param:** `client_record_id` — client-scoped listing is what the client page's tabs
do; search does not duplicate them.
**Response:** `PaginatedResponse[SearchMatch]`.
`result_type=client` remains a 422 by the enum, as today.

### 5.4 Backend structure

- `SearchItemRepository` (8 per-client dossier queries) → **deleted**, replaced by
  `SearchMatchRepository`:
  - **Rows query:** one `UNION ALL` across the eight type branches, each branch selecting the
    shared row shape + a rank literal, wrapped with `row_number() OVER (PARTITION BY
    result_type ORDER BY rank, anchor_date DESC) <= 5` for the grouped preview.
  - **Counts query:** one aggregate over the same `UNION ALL` for per-type exact totals
    (the chip numbers).
  - Expansion (`/search/items`) runs the single relevant branch with real
    `LIMIT/OFFSET` + count.
- Client resolution (`SearchResultRepository`) is unchanged.
- **Cost:** ~2 resolution queries + 2 match queries per request, replacing today's 19-query
  resolved search. Verified the same way the 19 was measured (`before_cursor_execute`).
- Every branch keeps its per-domain scoping exactly as the dossier queries had it:
  `ClientRecord.deleted_at IS NULL` join, per-type soft-delete (`deleted_at`,
  `is_deleted`/`superseded_by` for permanent documents), notifications unscoped beyond the
  client join.

### 5.5 What does not change

Stated so it is checked, not assumed:

- Router gating: `require_role(ADVISOR, SECRETARY)`.
- Per-domain soft-delete exclusions; records of soft-deleted clients never returned.
- `entity_route(...)` in `app/common/entity_links.py` stays the single owner of result URLs.
  Advance payments, documents, and client-scoped annual reports keep linking through their
  owning client's screen.
- No new authorization surface: search widens *what can be found*, never *by whom*.

---

## 6. Indexes and migration (D7)

**Phase 1 — btree only.** Exact match = equality; equality uses btree. Audit each phase-1
column (`\d` per table) and add plain btree where missing — expected candidates:
`permanent_documents.original_filename`, `tasks.title`, `notifications.recipient`,
`annual_reports.ita_reference`. Identifier and period columns are likely already indexed
(PKs, unique constraints, existing filters); add nothing that exists. One Alembic migration,
downgrade drops exactly what upgrade added.

**Phase 2 gate — `pg_trgm` + GIN.** No `ILIKE '%term%'` ships before this migration:
`CREATE EXTENSION IF NOT EXISTS pg_trgm` + GIN (`gin_trgm_ops`) indexes on every
phase-2 *contained* column (charge description, binder notes, filename, task title/description,
advance-payment notes, notification subject). Render's Postgres supports `pg_trgm`.

**Write cost is the price and it is acknowledged:** GIN index maintenance is paid on every
INSERT/UPDATE of the indexed columns, permanently. That is exactly why it is deferred until
phase 2 actually ships (D7) — and why phase 2 itself is gated on phase-1 measurement (§9).

Migration rules per `docs/backend/migrations.md`; no enum types are introduced, so no enum
downgrade concerns.

---

## 7. Frontend — `/search`

### 7.1 URL state

| Param | Meaning | Change |
|---|---|---|
| `search` | the term | now matches records too |
| `type` | expanded match group | now expands a *match* type, not a dossier group |
| `page` | the list being paged (§7.3) | unchanged rule |
| `page_size` | client-list page size | unchanged |

**Deleted:** `client_record_id`, `client_status`, `entity_type`, `binder_location_status`,
`binder_capacity_status` (and the already-dead `id_number`/`binder_number`). The MAT-64
normalization mechanism (`SEARCH_DROPPED_FILTER_KEYS`) strips all of them from old deep links
in one atomic URL write — the mechanism survives even though the params it was built for die.

Enum-guard parsing for the four filter enums (`searchUrlValues.ts` guards) is deleted with the
filters; the only guarded param left is `type` (`SearchMatchType` membership).

### 7.2 Page layout

- **Toolbar:** one text input. No advanced panel, no selects (decision: filters deleted).
  Debounced URL commit, draft/`isFetching` dimming, and autofocus behavior stay exactly as
  built.
- **Nothing typed:** the existing prompt `StateCard`.
- **Term typed — two labelled sections:**
  1. **לקוחות תואמים** — `SearchClientMatches` rows as today (name, ids, status,
     `matched_binder_numbers` explanation). Click **navigates** to `/clients/:id` (D1 — never
     automatically). Paginated when it is the active list (§7.3).
  2. **רשומות תואמות** — chip row ("הכל" + one chip per type with its exact total, only
     types with matches), then either the mixed preview (up to 5 per type, grouped with type
     headings) or, with a chip active, that type's full paginated list via `/search/items`.
     Every row shows: type icon, title, status badge, amount/date — and **the client's name +
     office number**. Row click navigates via the row's `href`.
- **No match at all** (zero clients, zero records): the single existing empty state with
  reset.
- Both sections can be non-empty at once — that is the point of the page. The old
  "chooser XOR feed" exclusivity died with the feed.

### 7.3 The `page` rule

`page` pages whichever list the user is in, keeping today's single-param contract:

- No `type` active → `page` pages the **clients** list; the matches section shows previews
  (no pager).
- `type` active → `page` pages the **expanded match list**; the clients section shows its
  first page without a pager. Changing/clearing `type` resets `page` to 1 (same atomic-write
  discipline as today's `handleTypeChange`).

### 7.4 Code-level changes

Deleted: `SearchFiltersBar` (+ its tests, and its "sanctioned FilterPanel exception" status —
the exception dies with it), `SearchSelectedClient`, `SearchItemFeed` (its chip row moves into
the new matches section), `utils/searchSelection.ts` (selection has no page state to derive),
the four enum guards in `utils/searchUrlValues.ts`, all `client_record_id` handling in
`useSearchPage`.

Kept/adapted: `SearchToolbar` (input only), `SearchClientMatches` (click = navigate),
`SearchItemRow` (gains the client identity line; renamed `SearchMatchRow`), `useSearchPage`
(one query for `/search`, one for expansion — same two-query structure it has now).

New: `SearchMatchesSection` (chips + grouped previews / expanded list).

Types come from `npm run gen:types` after the backend regenerates `openapi.json`; the
hand-written mirrors in `features/search/api/contracts.ts` follow the new models.

Tests: pure functions (term trimming, page rule, chip derivation) + SSR markup for the two
sections — the stack has no jsdom/RTL, so no `renderHook`; same test style the feature already
uses.

---

## 8. Frontend — client page binders tab (D8)

- Add `binders` to `CLIENT_DETAILS_TABS` + label ("קלסרים") in
  `CLIENT_DETAILS_TAB_LABELS`, positioned after `documents`.
- Tab content: paginated binder list consuming the **existing**
  `GET /api/v1/clients/{client_record_id}/binders` — no backend work.
- Columns follow the global `/binders` table minus the client column (the page is the
  client); `DataTable` per the table-primitive architecture.
- This closes the only real gap the dossier feed covered that tabs did not.

---

## 9. Phasing

**Phase 1 — exact only.** Everything in §3–§8 with tier-1 matching only: charge id, binder
number, full filename, normalized period, tax year, ita_reference, full title/recipient.
Low noise, already answers "חיוב 307" and "מצא את הקובץ הזה". Ships with the btree
migration only.

**Phase 2 — prefix and contained.** Gated on two things: (a) the trigram migration (D7), and
(b) **measurement of phase 1 on real data** — term distribution, zero-result rate, per-type
noise. If phase-1 exact matching turns out to cover the office's real queries, phase 2 may
shrink or not ship; that outcome is acceptable and cheap because no trigram cost was
prepaid.

Hebrew substring behavior over real data is verified as part of the phase-2 gate, before any
`ILIKE` ships.

---

## 10. Exposed dependencies (separate issues, not this spec's scope)

- **Tasks have no `client_record_id` filter on their list endpoint.** Search links to tasks
  via `entity_route` as today; the gap is the tasks domain's, and adding task search here must
  not silently fix it. Separate issue.
- **`/clients` does not search by office client number.** `/search` covers it (the term
  matches office number), which is consistent with the division of labor — but if users
  expect it on `/clients` too, that is a clients-domain issue.
- **Old `/search` deep links** carrying deleted params are normalized, not honored (§7.1) —
  no redirect layer, per ADR-0003.

---

## 11. Documents that change at implementation time

Per phase, in the same task as the code (non-negotiable rule):

- `docs/domains/search.md` ★ — full rewrite of the contract: two-phase model → matches model;
  the lines "the typed term does not filter items" and the `client_record_id`/selection
  rules stop being true; D-decisions preserved under Decisions.
- `docs/architecture/api-contracts.md` — new response models, deleted params.
- `backend/openapi.json` + `npm run gen:types` (regen both; `gen:types` overwrites
  `generated.ts` in place).
- `docs/frontend/page-refactor-status.md` — SearchPage row (MAT-63/64/87 notes updated to
  reflect the deletions as closed debt) + ClientDetails row for the binders tab.
- Migration entry under `docs/backend/migrations.md` conventions for each Alembic migration.
- Error-doc matrix only if a new error response is documented for the 422s
  (`docs/backend/error-doc-matrix.md`).

---

## 12. Acceptance criteria, grouped for Linear

Project **YM Tax CRM**, team **Matan malka**. One issue per block; blocks are ordered by
dependency.

### B1 — Backend: match contract + repository (phase 1)

- [ ] `SearchMatch`/`SearchMatchGroup`/`SearchMatchGroups` models exist;
      `SearchItem`/`SearchItemGroup`/`SearchItemGroups` deleted; `SearchItemType` renamed
      `SearchMatchType` (same 8 members, `client` still excluded).
- [ ] `GET /search` accepts only `search` (required, non-blank), `page`, `page_size`;
      the seven deleted params are gone from the signature and OpenAPI; blank term → 422.
- [ ] `GET /search/items` accepts `search` + `result_type` + pagination; `client_record_id`
      gone; returns `PaginatedResponse[SearchMatch]`.
- [ ] `SearchMatchRepository` implements the `UNION ALL` rows query (rank column,
      `row_number() ≤ 5` per type) + one counts query; dossier `SearchItemRepository`
      deleted.
- [ ] Term parser is a pure function with unit tests: period normalization (D5: `03/2026`,
      `3/2026`, `2026-03`), bare-integer classification (D4 identifier set, year heuristic
      1990–2100), whitespace rejection.
- [ ] Each branch matches exactly the §3.2 phase-1 columns — test per type, including the
      exclusions (no internal-id match: searching a task's DB id returns nothing).
- [ ] Scoping tests: soft-deleted record excluded per type; record of soft-deleted client
      excluded; superseded/deleted permanent documents excluded; roles enforced.
- [ ] Query-count test: resolved search request ≤ 4 queries (`before_cursor_execute`).
- [ ] `matches` and client resolution coexist in one response; `items` block gone.
- [ ] OpenAPI regenerated; `docs/domains/search.md` + `docs/architecture/api-contracts.md`
      updated in the same task.

### B2 — Backend: btree migration (phase 1)

- [ ] Index audit of every §3.2 phase-1 column documented in the migration message.
- [ ] Alembic migration adds only missing btree indexes; downgrade drops exactly them.
- [ ] No `pg_trgm`/GIN in this migration (D7).

### F1 — Frontend: `/search` page rework

- [ ] Toolbar = single input; `SearchFiltersBar`, advanced panel, and the four enum guards
      deleted; FilterPanel-exception note removed from docs that carry it.
- [ ] `useSearchPage`: no `client_record_id`/selection handling; `searchSelection.ts`
      deleted; two queries (`/search`, `/search/items` expansion) keyed on term + type +
      page.
- [ ] Two labelled sections; client click navigates to `/clients/:id`; **no
      auto-navigation under any match count** (D1).
- [ ] Match rows show client name + office number; row click follows `href`; chips show
      per-type exact totals; expanded type list paginates per the §7.3 `page` rule (clients
      pager hidden while `type` active; `type` change resets page atomically).
- [ ] Old-URL params (`client_record_id`, four filter enums, `id_number`, `binder_number`)
      stripped via the existing normalization effect in one URL write.
- [ ] Empty prompt, single no-match state, debounce dimming, autofocus — preserved.
- [ ] Tests: pure-fn + SSR markup (no renderHook); `npm run typecheck` + lint + arch:check +
      build green; `generated.ts` regenerated.
- [ ] `docs/frontend/page-refactor-status.md` SearchPage row updated (deletions recorded as
      closed debt).

### F2 — Frontend: client page binders tab (D8)

- [ ] `binders` tab in `CLIENT_DETAILS_TABS` + Hebrew label; routed like every other tab.
- [ ] Tab consumes existing `GET /clients/{id}/binders`; DataTable columns = global binders
      table minus client column; pagination.
- [ ] No backend changes.

### P2 — Phase 2: prefix + contained (gated)

- [ ] Gate 1: phase-1 usage measured on real data (term distribution, zero-result rate) and
      reviewed — go/no-go recorded in this document.
- [ ] Gate 2: `pg_trgm` extension + GIN migration on the §6 contained columns lands **before**
      any `ILIKE` query.
- [ ] Tiers 2–3 added to the rank column; per-tier ordering tests; Hebrew substring behavior
      verified on real data.
- [ ] `docs/domains/search.md` updated in the same task.
