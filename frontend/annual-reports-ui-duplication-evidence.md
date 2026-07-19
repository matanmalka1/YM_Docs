# Annual Reports UI Duplication — Evidence Review

Review date: 2026-07-16  
Scope: `frontend/src/features/annualReports/`  
Input: the attached “Investigation Report — frontend/src/features/annualReports/”  
Method: static inspection with repository-wide `rg` searches plus line-by-line comparison with the shared UI primitives.

This is a review artifact, not a source of truth. Binding frontend rules remain in `docs/frontend/architecture.md`, `docs/frontend/page-structure.md`, and `docs/frontend/ui-guidelines.md`.

## Result summary

| Claim | Verdict | Verified evidence / correction |
|---|---|---|
| Table UI uses `DataTable`; no raw `<table>` | Confirmed | Five components render `DataTable` directly; `AnnexDataPanel` delegates to `AnnexDataTable`. No raw `<table>` occurs in the feature. |
| Select UI uses the shared primitive; no raw `<select>` | Confirmed | No raw `<select>` occurs. Direct and wrapper usages ultimately render `Select`. |
| Status colors use semantic helpers / `Badge` | Confirmed | No `bg-green-*`, `text-green-*`, `bg-red-*`, or `text-red-*` occurs. Status rendering uses `getStatusVariant`, `Badge`, or semantic tone classes. |
| Currency and percentage display uses canonical formatters | Confirmed, with documented exemption | `formatCurrencyILS` occurs in 16 files and `formatPercent` in 4. The chart tick abbreviation in `MultiYearPLChart.tsx` is explicitly exempt under the frontend architecture rules. |
| No file exceeds 250 lines | Partially confirmed | No **component** exceeds 250 lines; `FinancialLineFormParts.tsx` is 231 lines. The feature as a whole does exceed 250 lines in `api/contracts.ts` (378) and `messages.ts` (273). |
| Four financial line forms share implementation | Confirmed | All four use the shared `FinancialLineFormParts.tsx` exports and `useFinancialLineForm` hooks. |
| Hand-rolled loading text | Confirmed, corrected count | 8 occurrences in 8 files, not 9 occurrences. |
| Hand-rolled non-table empty states | Confirmed | 4 occurrences in 4 files. |
| Local label/value rows overlap `DefinitionList` | Confirmed with API caveat | The structure overlaps `layout="stacked"`, but the local `Row` supports per-row muted/value styling that `DefinitionList` does not expose directly. |
| Raw card-like containers bypass `Card` | Confirmed, candidate list corrected | Seven reported locations exist, but only four are direct candidates for the current div-only `Card`. |
| Repeated chevron collapsibles | Confirmed | 4 annual-report implementations plus 2 cross-feature examples; no shared `Collapsible`/`Disclosure` primitive was found. |
| One-off badge and semantic-map findings | Confirmed | Both reported implementations exist and are isolated within this feature. |

## 1. Conventions that are holding

### 1.1 Tables

Repository search found no raw `<table>` under the feature. Direct shared-table consumers are:

- `components/annex/AnnexDataTable.tsx:82`
- `components/panel/ReportHistoryTable.tsx:92`
- `components/season/SeasonReportsTable.tsx:110`
- `components/shared/ClientYearComparisonModal.tsx:59`
- `components/tax/TaxBracketsTable.tsx:45`

`components/annex/AnnexDataPanel.tsx:25-38` renders `AnnexDataTable`, so it also routes tabular UI through the shared table indirectly.

### 1.2 Selects

No raw `<select>` was found. The shared `Select` is rendered directly in:

- `components/annex/ScheduleAddForm.tsx:58`
- `components/financials/FinancialLineFormParts.tsx:46`
- `components/shared/CreateReportModalParts.tsx:106`
- `components/statusTransition/TransitionDetailsForm.tsx:35`

The add/edit income and expense forms also consume shared select-field wrappers from `FinancialLineFormParts.tsx`; they do not implement native selects.

### 1.3 Status badges and semantic color

No hard-coded Tailwind green/red status classes were found. Representative evidence:

- `components/season/SeasonReportsTable.tsx:90` renders `Badge` with `getStatusVariant(r.status)`.
- `components/panel/ReportHistoryTable.tsx:67` uses the same status-to-badge mapping.
- `components/statusTransition/StatusTransitionPanelRoot.tsx:36` uses the same mapping for the current status.
- `components/annex/AnnexHeader.tsx:33-42` uses `Badge` and `semanticMonoToneClasses`.
- `components/panel/ReadinessCheckPanel.tsx:29-32` selects positive/negative semantic tones.

### 1.4 Currency and percentage formatting

`formatCurrencyILS` is imported/used in 16 feature files; `formatPercent` is used in 4. No `toLocaleString` occurs under the feature.

`components/financials/MultiYearPLChart.tsx:62` contains:

```tsx
tickFormatter={(value) => `₪${(Number(value) / 1000).toFixed(0)}K`}
```

This is not a violation. `docs/frontend/architecture.md` explicitly exempts chart-axis abbreviations such as `₪5K` when canonical formatters cannot express the required compact tick.

Other literal `₪` occurrences are labels, input suffixes, or explanatory copy rather than hand-rolled numeric display formatting, for example `CreateReportModalParts.tsx:8`, `messages.ts:204-206`, and `taxHelpers.ts:79`.

### 1.5 File size

The original wording needs narrowing:

- Largest component: `components/financials/FinancialLineFormParts.tsx` — 231 lines.
- Files above 250 lines elsewhere in the feature: `api/contracts.ts` — 378 lines; `messages.ts` — 273 lines.

Therefore, “no component exceeds 250 lines” is supported; “no file exceeds 250 lines” is not.

### 1.6 Shared financial-line form pattern

The four forms all delegate form state to the feature hook:

- `AddExpenseLineForm.tsx:11` and `EditExpenseLineForm.tsx:9` import `useExpenseLineForm`.
- `AnnualReportAddIncomeLineForm.tsx:3` and `EditIncomeLineForm.tsx:8` import `useIncomeLineForm`.

They also reuse the field/shell components exported by `FinancialLineFormParts.tsx`, which centralizes select configuration, inputs, errors, and submit/cancel UI. This is verified positive reuse.

## 2. Finding: hand-rolled loading states

Verdict: confirmed, but the correct count is **8 occurrences in 8 files**.

| Location | Evidence |
|---|---|
| `components/panel/ReadinessCheckPanel.tsx:21` | Conditional `<p className="text-sm text-gray-400 py-2">`. |
| `components/tax/TaxCreditsPanel.tsx:18` | Conditional gray loading `<p>`. |
| `components/annex/AnnexDataPanel.tsx:19` | Conditional gray loading `<p>`. |
| `components/tax/DeductionsTab.tsx:24-25` | Centered gray loading `<p>`. |
| `components/tax/TaxCalculationTab.tsx:42-47` | Centered gray calculating `<p>`. |
| `components/financials/AnnualPLSummary.tsx:47-49` | Conditional gray loading `<p>`. |
| `components/financials/IncomeExpenseTab.tsx:23-25` | Centered gray loading `<p>`. |
| `components/panel/AnnualReportFullPanel.tsx:54-59` | Centered gray loading `<div>`. |

Shared evidence: `src/components/ui/feedback/InlineState.tsx:16-43` supplies a centered icon, title, optional description, and action. The canonical helper index identifies `InlineState` for an inline empty/error/no-results state, while loading can also use the existing spinner/skeleton/page-loading components according to surface size.

Important nuance: `InlineState` is not necessarily an exact visual replacement for every compact one-line loader because it adds an icon and `py-10`. Reuse should select `Spinner`, a skeleton, `PageLoading`, or `InlineState` based on available space. `components/shared/ClientAnnualReportsTab.tsx:28` already demonstrates `PageLoading` reuse inside this feature.

## 3. Finding: hand-rolled empty states

Verdict: confirmed — **4 occurrences in 4 files**.

| Location | Evidence |
|---|---|
| `components/financials/AnnualReportFinancialSection.tsx:37` | Inline centered gray `<p>` when `isEmpty`. |
| `components/tax/DeductionsTab.tsx:48-52` | Inline centered gray `<p>` when expenses are empty. |
| `components/shared/ClientYearComparisonModal.tsx:56-59` | Modal swaps between a gray `<p>` and `DataTable`. |
| `components/annex/AnnualReportAnnexesTab.tsx:28-35` | A `Card` contains plain empty copy and the add form. |

The last case is not centered, so the report’s “each reimplements its own centered gray `<p>`” wording is too broad. It is still a locally composed empty state.

Correct table reuse is visible at:

- `components/panel/ReportHistoryTable.tsx:96` — `DataTable.emptyMessage`.
- `components/season/SeasonReportsTable.tsx:116` — `DataTable.emptyMessage`.

The report cited `IncomeExpenseTab.tsx:51,92` as `DataTable.emptyMessage` evidence, but those props belong to the feature-owned `AnnualReportFinancialSection`, not `DataTable`; that section renders the hand-rolled empty `<p>` at line 37.

## 4. Finding: label/value row duplication

Verdict: conceptually confirmed, but not a drop-in replacement.

`components/tax/TaxCalculationTab.tsx:17-27` defines `Row` with `<dt>/<dd>` inside a horizontal flex row. `SectionCard` wraps those rows in a `<dl>` at lines 29-35. The same file imports and renders `DefinitionList` at lines 3 and 61-76.

`src/components/ui/layout/DefinitionList.tsx:62-77` implements `layout="stacked"` with the same semantic structure and nearly the same layout: horizontal label/value alignment, vertical padding, and row dividers.

The local implementation additionally supports:

- per-row muted label styling;
- per-row value classes/tones;
- a custom section header and enclosing `Card`.

`DefinitionList` currently exposes one `valueClassName` for every row, not per-item value classes. `docs/frontend/ui-guidelines.md` also calls out this limitation. Reuse is possible by placing styled React nodes in each item’s `value`, but replacement is not purely mechanical.

`components/tax/TaxCreditsPanel.tsx:29-38` is a related label/description/value list. It has the same information pattern but different row composition, so it is a candidate for evaluation rather than direct duplication.

## 5. Finding: raw card-like containers

Verdict: seven reported locations exist, but the reusable-candidate assessment requires correction.

`Card` is used by 8 files in the feature. `src/components/ui/primitives/Card.tsx:5,59-81` accepts `HTMLAttributes<HTMLDivElement>` and always renders a `<div>`; it has no polymorphic `as` prop.

| Location | Element | Direct `Card` candidate? | Reason |
|---|---:|---:|---|
| `components/tax/TaxCreditsPanel.tsx:25` | `div` | Yes | Plain bordered white container. Header/body/footer details would need class mapping. |
| `components/annex/AnnexRowForm.tsx:29` | `form` | No | Replacing it would lose form semantics; `Card` cannot render a form. |
| `components/annex/ScheduleAddForm.tsx:56` | `div` | Yes | Plain bordered gray container. |
| `components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:88` | `section` | No | The report called this a plain div, but it is a semantic section. |
| `components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:119` | `section` | No | Same semantic mismatch. |
| `components/financials/AnnualReportVatAutoPopulateResultPanel.tsx:150` | `div` | Yes | `ResultMetric` is a plain bordered tile. Whether a full `Card` is visually appropriate remains a design judgment. |
| `components/tax/TaxCalculationTab.tsx:150` | `div` | Yes | Plain bordered summary container. |

The original “7 occurrences / 5 files” count is correct. The stronger claim that both VAT panel subsections are legitimate direct replacements is not supported by the current `Card` API.

The broader search also finds other intentional bordered surfaces (alerts, informational callouts, nested rows, and an `<li>` annex card). A rounded border alone is insufficient evidence that every such surface should become `Card`; semantic role and variant intent must be considered.

## 6. Finding: repeated chevron collapsibles

Verdict: confirmed.

| Location | State/toggle evidence | Revealed content |
|---|---|---|
| `components/shared/OverdueBanner.tsx:17,45-57` | Local `expanded`, toggle button, up/down chevrons, `aria-expanded`. | Report list switches between preview and full list. |
| `components/panel/AnnualReportOverviewTab.tsx:33,57-78` | Local `plExpanded`, toggle button, up/down chevrons. | P&L summary appears below a border. |
| `components/annex/AnnexCard.tsx:16,25-40` + `AnnexHeader.tsx:69` | Parent owns `isExpanded`; header toggles and swaps chevrons. | Annex data panel appears below a border. |
| `components/statusTransition/StatusTransitionPanelRoot.tsx:17,55-65,78-90` | Local `expanded`, toggle button, up/down chevrons. | Transition selector/form appears below a border. |

No file named for a shared `Collapsible` or `Disclosure` was found under `src/components/ui/`.

Cross-feature evidence supports a shared UI gap:

- `features/search/components/SearchFiltersBar.tsx:54` swaps chevrons for its controlled open state.
- `features/signatureRequests/components/list/SignatureRequestRow.tsx:45,99` owns expanded state and swaps chevrons.

These implementations do not have identical content or styling, so a new primitive should own disclosure behavior and accessibility—not impose one annual-report-specific visual container.

## 7. One-off findings

### 7.1 Badge-like span

Confirmed at `components/financials/FinancialLineRow.tsx:46-50`: a recognition rate below 100 renders as an `inline-flex`, rounded, warning-colored `<span>`. It is visually badge-like and does not use `Badge`. No second equivalent occurrence was found in this feature.

### 7.2 Local alert semantic maps

Confirmed at `components/panel/ReportAlertBanners.tsx:19-31`: `VARIANT_STYLES` and `ICON_STYLES` locally map error/warning/info/success to semantic background, border, text, and icon classes.

This overlaps conceptually with shared semantic color maps, but it is not necessarily a direct duplicate: the banner requires background + border + foreground combinations, while existing badge/mono-tone helpers may not provide the exact alert surface contract. It remains an isolated consistency review item, not evidence by itself for a new abstraction.

## 8. Corrected prioritized backlog

1. Standardize the 8 loading states using the appropriate existing feedback primitive for each surface.
2. Standardize the 4 non-table empty states; do not count `IncomeExpenseTab` as correct `DataTable.emptyMessage` reuse.
3. Evaluate `DefinitionList` reuse in `TaxCalculationTab`, accounting for per-row styling before removal of `Row`.
4. Consider `Card` only for the four div-based candidates; preserve `<form>` and `<section>` semantics.
5. Design a shared accessible disclosure primitive only as a cross-feature change, since six real consumers now demonstrate the behavior.
6. Leave the two one-off findings as low-priority consistency items unless broader duplication appears.

No critical behavior bug was identified by this static duplication review.
