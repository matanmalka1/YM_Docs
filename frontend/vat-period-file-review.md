## Scope

This file owns only:

- A review of the VAT periodic-file page (`VatWorkItemDetailPage`) and a proposed rebuild shape.

This file must not contain:

- Canonical VAT domain rules (see `docs/domains/vat.md`).
- Canonical frontend architecture or page-structure rules (see `docs/frontend/architecture.md`,
  `docs/frontend/page-structure.md`).
- Endpoint contracts (see `docs/architecture/api-contracts.md`).

Source of truth: review and backlog only — not source of truth for current behavior.

# VAT Periodic File (תיק תקופתי) — Page Review

Review of `frontend/src/features/vatReports` detail surface as of 2026-07-20. Each finding is
stated with its evidence and the proposed target shape.

**Status: P1 is done. P2–P8 are open.**

## Surface under review

| File | Role |
| --- | --- |
| `pages/VatWorkItemDetailPage.tsx` | page shell, tab switch |
| `hooks/useVatWorkItemDetailPage.ts` | page hook (tabs, header, filed banner) |
| `hooks/useVatWorkItemPage.ts` | two queries: work item + invoices |
| `components/detail/VatSummaryTab.tsx` | summary tab |
| `components/detail/VatInvoiceTab.tsx` | income tab **and** expense tab |
| `components/detail/VatHistoryTab.tsx` | audit trail tab |
| `components/list/VatWorkItemSummaryBar.tsx` | banners, progress, action bar, file modal |
| `components/shared/VatActionButtons.tsx` | four hand-written lifecycle buttons |

---

## P1 — Tax amounts computed in the browser — **DONE**

The original finding: `utils/vatBreakdown.ts` derived the per-category breakdown from raw invoices,
reverse-engineering gross VAT as `deductible / rate` and then re-deriving the displayed rate as
`deductible / grossVat`. Two float round-trips per category, producing a second set of numbers next
to the ones the backend actually files.

Resolved as designed:

- `VatInvoiceAggregationRepository.expense_breakdown()` groups expense invoices in SQL and returns
  `net_amount` / `gross_vat` / `deductible_vat` per category.
- `vat_report_enrichment.get_work_item_enriched()` wraps them in `VatBreakdownResponse`, exposed as
  the required `breakdown` field on `VatWorkItemResponse`.
- All six routes returning `VatWorkItemResponse` go through `serialize_work_item`, so the field is
  always populated.
- `utils/vatBreakdown.ts` is deleted; `VatSummaryTab` reads `workItem.breakdown` directly.

Two consequences worth recording:

**Credit notes now reduce income net.** The deleted client-side code summed income `net_amount`
unsigned; the aggregation applies `_signed_amount`, so `CREDIT_NOTE` rows subtract. Totals for
periods containing credit notes differ from what the page displayed before. This is the correction,
not a regression.

**Mixed deduction rates were collapsing.** The first implementation used
`func.max(deduction_rate)` as the row's rate. Where one category holds invoices at different stored
rates, the label described only part of the row's amounts. Fixed by making the rate part of the
`GROUP BY` key — a category with two rates now yields two truthful rows. Covered by
`test_expense_breakdown_keeps_mixed_deduction_rates_as_separate_rows`. The frontend already keyed
rows on `${category}-${deduction_rate}`, so no frontend change was needed.

**Still mixed-source (accepted).** `income_net` and `total_expense_net` come from the stored
`total_output_net` / `total_input_net` columns, while `total_gross_vat` and the category rows are
computed live per read. They agree as long as `recalculate_totals` has run — it is called on invoice
create, update, and delete. Not unified deliberately: post-FILED totals are snapshots and must not
be recomputed (`vat_work_item.py:81`), so switching the stored values to live computation would make
a filed period's summary drift from what was filed.

## P2 — Filing state is managed by three ad-hoc workarounds

Filing is irreversible, but there is no single "this work item is mutating" state. Three separate
mechanisms stand in for it:

1. `isFilingPending` — a boolean created in `useVatWorkItemDetailPage`, passed page → summary bar →
   file modal → back up, purely to disable invoice editing (`VatInvoiceTab.tsx:22`).
2. `ACTION_CHANGE_COOLDOWN_MS = 1500` in `useVatWorkItemActions.ts` — a timer that blocks all
   actions after any status change, because the button set re-renders under the cursor and a second
   click could land on a different action.
3. `queryClient.setQueryData(..., { ...prev, status: 'filed' })` in `useFileVatReturn.ts:21` — a
   hand-patched cache entry that sets one field, before the real invalidation lands.

Each is a symptom of the same gap. Target: one in-flight state per work item. While any lifecycle
mutation is pending, every action and every invoice mutation is disabled, and the UI waits for the
server response rather than patching a field. The cooldown timer, the pending flag, and the manual
`setQueryData` all disappear.

## P3 — `available_actions` arrives from the server, then is re-implemented client-side

The backend sends `available_actions`. The frontend translates it through four predicate helpers
(`canMarkMaterialsComplete`, `canMarkReadyForReview`, `canFile`, `canSendBack`) into four
hand-written `<Button>` blocks in `VatActionButtons.tsx`, each repeating the same
variant/size/isLoading/disabled props, plus three matching handlers in `useVatWorkItemActions.ts`.

Adding one lifecycle action means editing three files. Target: a declarative action map keyed by the
server's action key (`label`, `icon`, `variant`, `mutation`), rendered in a loop. The predicates
collapse into `available_actions.includes(key)`.

## P4 — Income and expense are one entity rendered as two tabs

`VatInvoiceTab` is parameterized by `invoiceType` and then branches on it for title, border color,
empty message, and whether the category filter renders (`VatInvoiceTab.tsx:26-37`). The page mounts
it twice with near-identical prop lists (`VatWorkItemDetailPage.tsx:71-88`).

An invoice is one entity with a `type` field. Target: a single **transactions** tab with a
type segment control, matching how the period is actually worked (entries go in, then get reconciled
across both sides).

## P5 — Filtering and counting happen in memory over an unbounded list

- `GET /work-items/{id}/invoices` is non-paginated by design (`vat_routes_data_entry.py:77`), and
  supports an `invoice_type` query param that the frontend does not use.
- The category filter is applied with `Array.filter` over the full list (`VatInvoiceTab.tsx:28`).
- Tab badge counts are derived from the same full list
  (`useVatWorkItemDetailPage.ts:35-36`), so they render as `0` until invoices load, then jump.

Target: `type` and `category` become server params with pagination; counts come from the work-item
response so badges are correct on first paint.

## P6 — Page composition drifts from the canonical page-hook contract

- `useVatWorkItemDetailPage` returns a partial slot set; the page still branches with four
  `activeTab === '…'` checks.
- `setTab` calls `setSearchParams(tab === 'summary' ? {} : { tab })`
  (`useVatWorkItemDetailPage.ts:31`), which **discards every other URL param**.
- `VatSummaryTab` reaches for `useSearchParamFilters` itself in order to switch tabs
  (`VatSummaryTab.tsx:11`). A leaf component mutating page navigation state.
- `VatWorkItemMetaStrip` fires its own query via `useActiveVatBinder`
  (`VatWorkItemMetaStrip.tsx:10`). A presentational strip owns a data dependency.

Target: tabs become child routes (`/tax/vat/:id/:tab?`), the shell renders `<Outlet />`, navigation
is passed down as a callback, and the binder arrives in the work-item response.

## P7 — Loading, empty, and error states use three different vocabularies

Within `VatHistoryTab.tsx` alone: loading is a bare `<p>` (line 41), error is `InlineState` with a
retry action (line 44), empty is another bare `<p>` (line 52). The page shell hand-assembles two
`TableSkeleton` blocks (`VatWorkItemDetailPage.tsx:22-27`).

The page-level error branch (`VatWorkItemDetailPage.tsx:29`) renders a bare `<Alert>` with no retry
and no header — breadcrumbs and page title vanish, so the user loses all context and any way back.

Target: the sanctioned section-loading variant — header and meta stay mounted, the section owns its
loading/error/empty, and error offers retry. See `docs/frontend/page-structure.md`.

## P8 — Mutations are hand-rolled around an existing helper

`useVatWorkItemActions.run()`, `runMutationWithFeedback` in `useVatInvoiceMutations.ts`, and the
manual `setIsLoading` in `useFileVatReturn.ts` each re-implement try/catch → toast → invalidate.
`useMutationWithToast` already exists for exactly this shape.

Note: filing legitimately needs result inspection (`fileVatReturn` returns `boolean` to close the
modal), which is one of the documented cases the helper cannot express. The invoice mutations and
the three lifecycle actions are not.

---

## Rebuild shape

If the page were built from scratch:

**Server.** One response describes the whole periodic file: header (client, period, deadline,
binder, assignee), lifecycle (`status`, `available_actions`, filed info), computed `breakdown`,
and invoice counts. One request, one render, no arithmetic in the client.

**Routing.** `/tax/vat/:id/:tab?` with child routes for `summary`, `transactions`, `history`. Tab
identity lives in the URL path, not in a search param that overwrites its neighbors, and each tab
loads lazily.

**Shell.** Header + meta strip + action bar + `<Outlet />`. No `activeTab` branching.

**Actions.** One action bar driven by `available_actions`, one mutation hook per action built on
`useMutationWithToast`, one in-flight lock covering actions and invoice editing alike.

**Transactions.** One tab, type as a segment, `type` and `category` as server params, paginated.

**Page hook.** The canonical grouped slots (`status`, `headerProps`, `actions`, `tabs`, `modals`),
with the page as pure slot composition.

## Suggested order

1. ~~**P1** — move VAT math to the server.~~ Done.
2. **P2 + P3** — single in-flight state and a declarative action list. These two are one change;
   doing either alone leaves the other's workarounds in place.
3. **P7** — state vocabulary, including keeping the header on error.
4. **P4 + P5** — merge the invoice tabs and move filtering to the server.
5. **P6** — route-based tabs and page-hook contract alignment.

P6 is the largest diff and the smallest behavior change; it should not be scheduled ahead of P1–P3.
