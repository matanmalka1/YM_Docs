# Frontend code-health audit — implementation ledger

Implementation started: 2026-07-22

Status values: `pending`, `completed`, `rejected`, `blocked`.
Cross-references `DU-01`–`DU-03`, `DU-10`, and `SQ-23` are represented by their canonical findings and are not counted twice.

| Finding | Severity | Status | Evidence / implementation note |
|---|---|---|---|
| DC-01 | P3 | completed | Deleted ignored `notifications/.DS_Store`; consumer proof repeated with import/file inventory. |
| DC-02 | P2 | completed | Removed three unused annual-report transports and cascade-only endpoint/key/DTO members; retained live `AdvancesSummary`. |
| DC-03 | P2 | completed | Removed obsolete transition endpoint and unused annual query-key members. |
| DC-04 | P3 | completed | Removed unused `CURRENT_YEAR`. |
| DC-05 | P2 | completed | Removed dead binder open-list/intake-update APIs and their private contracts, keys, and endpoints. |
| DC-06 | P3 | completed | Removed orphan binder key and narrowed binder barrel exports. |
| DC-07 | P2 | completed | Removed unused business restore API/endpoint and six unused client route builders after route/import revalidation. |
| DC-08 | P3 | completed | Removed unused client keys (`taxProfile`, `firstBusiness`) and narrowed client public surface. |
| DC-09 | P2 | completed | Removed three unused VAT transports and cascade-only request/endpoint members. |
| DC-10 | P3 | completed | Removed unused category-table label clone. |
| DC-11 | P2 | completed | Removed unused annual-report document list API, endpoint, and key. |
| DC-12 | P2 | completed | Removed unused authority-contact detail API/key and its test-only assertion. |
| DC-13 | P3 | completed | Removed unused `usersApi.getById`; retained shared by-ID endpoint used by update. |
| DC-14 | P3 | completed | Removed five orphan query-key roots across correspondence, signatures, tasks, dashboard, and timeline. |
| DC-15 | P3 | completed | Removed unused core health/info endpoint constants. |
| DC-16 | P3 | completed | Narrowed charge API and feature barrels without deleting live source declarations. |
| DC-17 | P3 | completed | Removed export modifiers from file-local URL helpers and search composition types. |
| DC-18 | P3 | completed | Removed obsolete `RowActionButton.danger`; preserved live `RowActionItem.danger`. |
| DC-19 | P2 | completed | Removed unreachable aging-report embedded prop/branches. |
| DC-20 | P2 | rejected | Current code now has a visible, URL-backed historical-year selector required by SQ-31; removing it would regress refresh/back/share behavior, contradicting the audit's historical snapshot. |
| SQ-01 | P1 | completed | Rebuilt checker with TypeScript resolution for aliases, lazy/re-export/type-only edges and added architecture fixtures; strict graph is clean. |
| SQ-02 | P1 | completed | Moved connected client picker/search into clients; generic `FilterPanel` now accepts typed custom slots. |
| SQ-03 | P1 | completed | Replaced own-feature root imports with local paths; fixture-enforced by repaired checker. |
| SQ-04 | P1 | completed | Established task contract-only public API and removed tasks/workQueue dependency-direction bypass. |
| SQ-05 | P1 | completed | Broke clients/businesses and binders/VAT/client graph cycles incrementally; strict checker reports zero SCC violations. |
| SQ-06 | P2 | completed | Removed annual-to-timeline invalidation edge and private label suppressions; timeline uses contract-only public maps. |
| SQ-07 | P1 | completed | Moved connected `ClientSidebar` directory/query/grouping to clients ownership; router uses feature root. |
| SQ-08 | P2 | completed | Moved `useBusinessesForClient` to clients and removed unused auto-select API. |
| SQ-09 | P1 | completed | Moved client lifecycle policy from app utility into clients feature. |
| SQ-10 | P2 | completed | Moved reusable client advance-payment tab from pages to feature components. |
| SQ-11 | P1 | completed | Replaced fake custom native facade with a real `<select>` preserving refs/events/required/disabled/form behavior; added component tests. |
| SQ-12 | P2 | completed | Removed five unused `StatsCard` variants and synchronization animation; added static/loading/interactive tests. |
| SQ-13 | P1 | completed | Split settings parser/grouping/tables and introduced focused `useTaxCalendarSettingsPage`; added utility tests. |
| SQ-14 | P1 | completed | Dashboard page hook now owns grouped, prewired modal workflow props/actions instead of leaking setters. |
| SQ-15 | P1 | completed | Added one client-details page hook and document-owned signal query with explicit visible error policy/canonical route. |
| SQ-16 | P1 | completed | Split VAT URL filters and mutation/dialog actions behind the existing page composer. |
| SQ-17 | P1 | completed | Added honest binder input/output types and pure payload builder; replaced sync effects/silent lookup with explicit handlers/errors and tests. |
| SQ-18 | P2 | completed | Removed dependency suppression via pure URL initialization helper; tested missing/explicit/all/reset cases. |
| SQ-19 | P3 | completed | Inlined notification bell state and deleted one-consumer hook/export. |
| SQ-20 | P2 | completed | Inlined typed detail query into annual-report hook and deleted generic wrapper/casts. |
| SQ-21 | P2 | completed | Removed `SelectOptions` wrapper and migrated fields to typed RHF controllers. |
| SQ-22 | P3 | completed | Removed signature status and document search aliases; consumers use canonical constants. |
| SQ-24 | P3 | completed | Removed unused role capabilities and business auto-select option. |
| SQ-25 | P3 | completed | Moved one-domain table-selection hook under charges. |
| SQ-26 | P2 | completed | Retired taxProfile facade; advanced payments owns the read-only client configuration query and dead write path is deleted. |
| SQ-27 | P2 | completed | Split tax-calendar contracts/endpoints/keys/transport and removed wildcard monolith surface. |
| SQ-28 | P2 | completed | Moved annual display labels/variants/progress configuration out of API and published immutable cross-feature contracts. |
| SQ-30 | P2 | completed | Removed reports formatter exclusion and formatted the full reports feature. |
| SQ-31 | P2 | completed | Report filters/pagination are URL-backed and errors normalized; added URL-state tests. |
| SQ-32 | P3 | completed | Removed redundant per-query default stale-time declarations while preserving deliberate policy overrides. |
| SQ-33 | P1 | completed | Defined one settings default range used by initialization/reset; exact semantics tested. |
| SQ-34 | P2 | completed | Removed unproduced timeline future/deadline shape and unreachable group branch; added grouping tests. |
| CT-01 | P0 | completed | Corrected VAT PATCH presence/null serialization, preserving document type and allowing counterparty clears; serializer matrix tests added. |
| CT-02 | P0 | completed | Invoice-create result now surfaces backend `ceiling_warning`; true/false feedback regression test added. |
| CT-03 | P0 | completed | Annual expense create omits untouched recognition rate so backend category default applies; override/default tests added. |
| CT-04 | P0 | completed | Recognition ratios use ratio-aware percentage formatting (`0.75` → `75%`); formatter regression covered. |
| CT-05 | P0 | completed | Removed frontend tax-credit engine; backend returns authoritative breakdown/credit/donation results with backend tax-engine tests. Canonical DU-01. |
| CT-06 | P0 | completed | Merged advance batches now sum `paid` and `not_paid`; regression tests cover workflow totals. |
| CT-07 | P0 | completed | Calendar groups complete only when no open child remains; mixed/overdue/done tests added. |
| CT-08 | P0 | completed | Canceled reports remain in season aggregation and never count in progress; frontend/backend regressions added. |
| CT-09 | P0 | completed | Secretary notification capability aligned with backend route; allowed/denied role matrix tested. |
| CT-10 | P0 | completed | Secretary signature management capability aligned with backend; role matrix tested. |
| CT-11 | P0 | completed | Secretary VAT creation capability aligned with backend; role matrix tested. |
| CT-12 | P0 | completed | VAT invoice deletion requires advisor-only capability plus backend action; denied/allowed tests added. |
| CT-13 | P0 | completed | Charge visibility/mutation capabilities aligned with backend routes and action descriptors; frontend role and backend action tests added. |
| CT-14 | P0 | completed | Terminal annual reports no longer show overdue alongside completion; open/terminal helper and backend tests added. |
| CT-15 | P1 | completed | Added authenticated backend VAT deduction metadata from canonical tax-rule registry; removed frontend legal table and render backend labels/rates/conditions. Canonical DU-02. |
| CT-16 | P1 | completed | Backend owns annual `is_overdue`/`days_until_deadline` with Israel-date/open-status semantics; frontend local copies removed. Canonical DU-03. |
| CT-17 | P2 | completed | Corrected annual status/deadline mutation response contracts to actual endpoint shapes. |
| CT-18 | P2 | completed | Client create no longer derives/sends backend-owned `id_number_type`; mapper test path retained. |
| CT-19 | P2 | completed | User audit DTO/rendering now preserves actor and target display names. |
| CT-20 | P3 | completed | Tax-calendar group item contract now includes backend `id_number`. |
| CT-21 | P3 | blocked | Backend exposes `data_version` but no implementation increments or consumes it; product/API must decide optimistic-concurrency semantics before a safe UI behavior exists. |
| CT-22 | P3 | completed | Narrowed charge-create payload to fields accepted/used by endpoint. |
| CT-23 | P0 | completed | Dashboard uses canonical `/work-queue` route constant; rendered-link regression added. |
| CT-24 | P0 | completed | Signature payload now preserves supplied signer phone; serializer test added. |
| CT-25 | P0 | completed | Signature schema mirrors backend title/name/description limits; boundary tests added. |
| CT-26 | P1 | completed | Backend returns structured warning code/fields; frontend exhaustive localization no longer parses English prose; tests on both layers. |
| CT-27 | P1 | completed | Backend dashboard links now use canonical entity-link builder; service route-contract tests added. |
| CT-28 | P2 | completed | Added request-only annual detail PATCH type; server-owned response fields cannot be submitted. |
| CT-29 | P2 | completed | Added backend notification metadata endpoint and frontend metadata query for form/list labels; removed duplicate trigger tables. Canonical DU-10. |
| DU-04 | P2 | completed | Consolidated annual status labels/stage order and included canceled state; typed display tests added. |
| DU-05 | P2 | completed | Consolidated annual client-type short labels and derived consumers/options. |
| DU-06 | P2 | completed | Centralized VAT reporting-frequency type/labels in `types/vatReporting.ts`. |
| DU-07 | P2 | completed | Extracted domain-specific `ChargeCoreFields`; create/edit retain shared schema/payload owner. |
| DU-08 | P2 | completed | Extracted `ChargesWorkspaceBody` shared by global/client hosts while retaining host navigation shells. |
| DU-09 | P2 | completed | Extracted `BindersWorkspaceBody` shared by global/client hosts while retaining host shells. |
| UW-06 | P2 | completed | Replaced sole root UI barrel imports with deliberate area imports and deleted mega-barrel. |

## Baseline

- Frontend worktree: clean.
- Backend worktree: clean.
- Docs worktree: pre-existing untracked `research/frontend-code-health-audit/` directory.
- `npm run typecheck`: passed.
- `npm run lint`: passed.
- `npm run test`: passed, 30 files / 117 tests.
- `npm run arch:check:strict`: passed, with the known pre-repair `@/` alias blind spot tracked by SQ-01.

## Final accounting

- Completed: 86 (`P0` 17, `P1` 18, `P2` 33, `P3` 18).
- Rejected after revalidation: 1 (`DC-20`, P2).
- Blocked by a backend/product decision: 1 (`CT-21`, P3).
- Pending: 0.

## Final verification

- Frontend `npm run check:strict`: passed; typecheck, lint, Prettier, strict architecture, 46 Vitest files / 171 tests, and Knip all completed successfully. Knip reports only the two intentional `@auditContract` tag hints.
- Frontend `npm run build`: passed; 4,254 modules transformed. Vite reports the pre-existing large-main-chunk advisory.
- Backend `.venv/bin/ruff format --check app tests tax_rules_config`: passed for 910 files.
- Backend `.venv/bin/ruff check app tests tax_rules_config`: passed.
- Backend focused contract suite: passed, 78 tests.
- `git diff HEAD --check`: passed in frontend and backend.
- Effective frontend inventory relative to `HEAD`: 48 added, 242 modified, 18 moved, and 19 deleted paths (327 paths total). Source excluding regenerated `src/types/generated.ts`: 3,561 lines added / 3,772 removed (net 211 removed).
- Effective backend inventory relative to `HEAD`: 27 modified paths. Excluding regenerated `openapi.json`: 249 lines added / 73 removed.
