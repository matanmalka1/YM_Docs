# Frontend code-health audit — executive summary

Audit date: 2026-07-22
Scope: `frontend/src/**`, frontend routing/configuration/tooling, related frontend tests, and the backend schemas/services needed to validate shared contracts. Production code was not modified.

## Overall assessment

The frontend is **fully reachable but structurally entangled**. Static reachability from `src/main.tsx` covers all 840 production TS/TSX/CSS files, and Knip reports no unused source file. The deletion opportunity is therefore mostly inside live files: unused API methods, query-key members, compatibility props, over-exported symbols, and branches of over-configured primitives.

Correctness risk is materially higher than the dead-file count suggests. The audit confirmed contract defects in VAT invoice editing, annual-report tax/expense calculations, permissions, aggregate status/count logic, and signature-request creation. It also found five runtime strongly connected components; the largest contains 159 modules. The mandatory architecture checker reports green because it does not resolve the project-standard `@/` aliases.

The strongest parts of the codebase are the broad use of typed DTOs, explicit query keys, shared UI primitives, feature-level message catalogs, and an already-converging page/controller convention. The weakest areas are handwritten adapters/calculations at backend boundaries, cross-feature barrel topology, custom form/select contracts, and a handful of large orchestration hooks/pages without focused tests.

## Scope and validation

| Measure | Result |
|---|---:|
| Files inspected under `frontend/src` | 870 |
| Production TS/TSX/CSS files | 840 |
| Vitest files | 28 |
| Other inspected files | 1 README, 1 ignored `.DS_Store` |
| Production files reachable from `main.tsx` | 840 / 840 |
| Confirmed dead production files | 0 |
| Immediate file-level deletion candidates | 1 (`.DS_Store`) |
| Tests executed | 28 files / 107 tests, all passing |
| Current clone scan | 16 clone groups, about 0.5% |
| Runtime strongly connected components | 159, 4, 2, 2, and 2 modules |

The import graph resolves TypeScript aliases, re-exports, CSS imports, and `import()`. API/query-key members were checked for direct, bracket, destructured, renamed, test-only, and cascade-only consumers. Routes, feature barrels, related tests, git-visible replacement history, OpenAPI, backend request/response schemas, permission decorators, services, and domain documentation were inspected where relevant.

The frontend repository currently also contains an unrelated untracked `.claude/worktrees/table-refactor/` copy outside `src`; a full Knip run enumerates that copy as unused noise. JSON filtering of the current run confirms `unused_src_files: []` for the canonical `frontend/src` tree. The audit did not modify or classify the external worktree copy.

## Finding count

The counts below use one canonical finding per problem. A finding may be cross-referenced from multiple reports without being counted twice. `SQ-23` is the same obsolete `RowActionButton.danger` surface as `DC-18` and is counted only as `DC-18`.

| Severity | Findings |
|---|---:|
| P0 | 17 |
| P1 | 18 |
| P2 | 34 |
| P3 | 19 |
| **Total** | **88** |

Confidence is recorded per finding. The majority are `Confirmed`; the limited runtime-dependent cases are called out explicitly rather than promoted to confirmed behavior.

| Canonical ledger | P0 | P1 | P2 | P3 | Total |
|---|---:|---:|---:|---:|---:|
| Dead/member-level findings (`DC-01`–`DC-20`) | 0 | 0 | 9 | 11 | 20 |
| Structural findings (`SQ-01`–`SQ-25`, excluding the `DC-18` duplicate) | 0 | 13 | 7 | 4 | 24 |
| Contract findings (`CT-01`–`CT-29`) | 17 | 4 | 5 | 3 | 29 |
| Additional wrapper/abstraction/dirty/boundary/duplication findings | 0 | 1 | 13 | 1 | 15 |
| **Total** | **17** | **18** | **34** | **19** | **88** |

## Highest-risk areas

1. **VAT invoice edit/create contracts.** The shared serializer sends presence/null combinations that can make every income edit fail with 422, clear `document_type`, and prevent clearing a nullable counterparty. A backend-generated OSEK-PATUR warning is also discarded.
2. **Annual-report tax correctness.** New vehicle/communication expenses override the backend's 75%/80% statutory defaults with 100%; ratios are rendered as `0.75%`; and `TaxCreditsPanel` reimplements the tax engine with wrong year constants and semantics.
3. **Role/capability drift.** Secretary workflows supported by backend routes are hidden for notifications, signature requests, VAT creation, and charges, while VAT invoice deletion is over-granted by a broad frontend flag.
4. **Aggregate state drift.** Advance-payment merged counts, tax-calendar group state, canceled annual-report season totals, and terminal-report overdue banners disagree with authoritative backend semantics.
5. **Architecture validation/topology.** `arch-check --strict` ignores alias imports, while live barrels and shared domain-connected components create a 159-module SCC plus four smaller cycles.
6. **Form primitive contract.** `Select` claims native `<select>` attributes, drops `required` and React Hook Form refs, and emits asserted synthetic event-shaped objects across 65 instances.

## Largest deletion opportunities

- Remove 12 unconsumed API methods and their cascade-only endpoint/contract members.
- Remove 14 unconsumed query-key members.
- Narrow 20 unused public export/type surfaces; only two underlying declarations are wholly dead.
- Delete obsolete report configuration (`AgingReportView.embedded`, controlled annual-status year/setter), `RowActionButton.danger`, two unused `useRole` capabilities, `useBusinessesForClient.onAutoSelect`, and five unused `StatsCard` feature branches.
- Inline/delete the one-consumer `useNotificationBell`, `useDetailQuery`, and `SelectOptions` wrappers; remove two pure aliases and the six-line root UI mega-barrel.

Estimated low-risk removable source volume is **approximately 280–340 LOC**, plus the 6,148-byte `.DS_Store`. This excludes migrations whose final files become deletable only after ownership is moved, such as the residual `taxProfile` facade.

## Worst abstractions

- `Select` / `SelectDropdown`: a custom listbox behind a false native-control API.
- `ClientSearchInput` plus `FilterPanel`'s `client-picker`: shared infrastructure that owns client API behavior and anchors the largest cycle.
- `useReceiveBinderDrawer`: form defaults, four domain queries, synchronization effects, payload construction, mutation side effects, and navigation in one 316-line hook.
- `TaxCalendarSettingsPage` plus its raw-query hook: validation, parsing, grouping, columns, permissions, mutation, and rendering in one 361-line page.
- `useVatWorkItemsPage`: a 267-line page hook that became the new controller dump after the page was simplified.
- Broad root barrels: API/contract imports transitively load pages and connected components, creating clients/businesses and binders/VAT/client cycles.
- The residual `taxProfile` facade: a renamed client-detail query retained after the original feature was removed, with a mutation output no caller uses.

## Biggest duplication clusters

Ten meaningful clusters are documented in [04-duplication.md](./04-duplication.md). The most expensive are frontend copies of backend tax-credit rules, VAT deduction law, and annual deadline semantics. The largest UI duplication worth consolidating is the charge create/edit core field block. Client/global charges and binder workspaces repeat orchestration worth extracting into domain-specific workspace bodies. Small stat-card composition, annual income/expense shells, and short unrelated listbox patterns were reviewed and intentionally left as acceptable duplication.

## Recommended cleanup order

1. Fix the P0 payload/calculation defects: VAT edit serialization, annual expense defaults, percentage display, and local tax-credit calculations.
2. Correct permission/capability drift with backend-aligned role-matrix tests.
3. Fix the small P0 navigation, signature payload/validation, aggregate count/state, and contradictory alert defects.
4. Repair `arch-check` alias resolution in report-only mode, then remove the 14 self-barrel imports and break concrete cycles one at a time.
5. Execute proven member-level deletions and wrapper inlining.
6. Replace the false native `Select` contract and migrate form callers with interaction coverage.
7. Move connected client search/sidebar/business-state code to the clients/business owner.
8. Split binder receive, VAT work-items, tax-calendar settings, dashboard, and client-details orchestration along focused workflow boundaries.
9. Consolidate domain-specific UI duplication, starting with charge form fields and client/global workspace bodies.
10. Coordinate backend contract additions for tax/VAT rule metadata, canonical deadline state, structured tax-calendar warnings, and route descriptors.

The task-level sequence, independence, tests, and migration risk are in [08-cleanup-plan.md](./08-cleanup-plan.md). The complete one-row-per-file disposition is in [09-file-inventory.md](./09-file-inventory.md).

## Static limitations

- Runtime interaction tests are still required to determine the visible impact of dropped `Select` refs/`required`, late binder-business responses overwriting form values, and midnight/DST deadline boundaries.
- Static analysis proves that the listed API methods have no shipped frontend consumer, but cannot establish whether a product team plans an unreleased screen.
- Charge authorization is internally contradictory on the backend: routes/docs/API tests allow secretaries while action-descriptor code/tests suppress them. Product intent must be resolved before changing every charge action.
- Direct navigation and all rendered modal/table states were not browser-tested. Import reachability and schema/consumer evidence do not depend on that limitation.
- The purpose of the backend's unused annual-annex `data_version` field could not be established from current reads or writes.
