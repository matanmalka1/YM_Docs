# Frontend code-health audit — duplication

Audit date: 2026-07-22
Scope: production code under `frontend/src/**`, with backend sources inspected where the duplicated concept is backend-owned. Production code was not modified.

## Method and result

A current clone scan reports **16 clone groups and approximately 0.5% duplicated lines**. The checked-in `frontend/jscpd-report` is older (2026-06-28) and names files that have since moved or disappeared, so every cluster below was revalidated against current imports, consumers, tests, and backend sources.

Most textual clones are harmless. The ten clusters below are included because they duplicate business rules, can produce behavioral drift, or repeat enough domain-specific orchestration that a local shared body would materially reduce maintenance risk. They are not ten additional findings in the audit-wide count when they cross-reference a contract finding.

## DU-01 — frontend tax-credit calculation duplicates the backend tax engine

**Severity/confidence:** P0 · Confirmed · same root problem as `CT-05` in [07-contract-leaks.md](./07-contract-leaks.md).

- **Canonical rule or behavior:** yearly credit-point values and tax-credit/deduction semantics belong to `backend/tax_rules_config/app/tax_rules/financials/constants_2024.py:111-119`, `constants_2025.py:173-181`, `constants_2026.py:185-194`, and `backend/app/annual_reports/annual_report_tax_engine.py:54-105`.
- **Duplicated implementations:** `frontend/src/features/annualReports/constants/taxConstants.ts:17-25`, `utils/taxHelpers.ts:27,48-99`, and the server result rendering in `components/tax/TaxCalculationTab.tsx:47-66,90-121`.
- **Behavioral differences:** the frontend says 2025/2026 are both 3003 while the backend says 2820/2904; the frontend converts pension contribution into a 4.5% “credit” and labels it as a training fund, while the backend treats the stored amount as a direct deduction; frontend rows omit donation credit. `TaxCreditsPanel` and `TaxCalculationTab` can therefore disagree inside one report.
- **Likely source of truth:** the backend tax engine and versioned tax-rules package.
- **Consolidation recommendation:** delete `getCreditPointValue`, `buildCreditRows`, `sumCreditRows`, and their local constants. Render the backend result; if the UI needs labeled components, add a server-provided breakdown DTO.
- **Is consolidation worth doing?** Yes, mandatory. This is persisted tax correctness, not cosmetic duplication. It requires backend-response coverage and frontend rendering tests.

## DU-02 — VAT deduction law is copied into frontend constants

**Severity/confidence:** P1 · Confirmed · same root problem as `CT-15`.

- **Canonical rule or behavior:** VAT deduction categories, rates, and conditional treatment live in `backend/tax_rules_config/app/tax_rules/vat_deduction.py:5-180` and are consumed by `backend/app/vat/vat_data_entry_common.py:92-99`.
- **Duplicated implementations:** `frontend/src/features/vatReports/constants/vatConstants.ts:64-147` owns `EXPENSE_CATEGORIES`, labels, and a 24-entry `DEDUCTION_RATES` table; `utils/viewHelpers.ts:31-38` converts that table into user-facing legal/tax hints; `components/form/VatInvoiceAddForm.tsx:37-41,167-171` renders them.
- **Behavioral differences:** backend partial deduction is 0.6667 while the frontend uses `2/3`; labels differ for vehicle maintenance/leasing/insurance; the backend notes vehicle treatment depends on classification, while the frontend presents a fixed percentage.
- **Likely source of truth:** backend tax-rules configuration.
- **Consolidation recommendation:** expose category/rate/condition metadata or a computed preview from the backend, retaining only explicitly UI-owned copy in React.
- **Is consolidation worth doing?** Yes. Tax rules change independently of frontend deployments; medium migration risk and backend coordination are justified.

## DU-03 — annual-report deadline and terminal-state logic has three incompatible copies

**Severity/confidence:** P1 · High confidence; the terminal-state defect is confirmed (`CT-14`), while midnight/DST differences need runtime verification (`CT-16`).

- **Canonical rule or behavior:** `backend/app/annual_reports/repositories/annual_report_report_lifecycle_repository.py:38-54` defines overdue using server time and open statuses.
- **Duplicated implementations:** `frontend/src/features/annualReports/utils/sharedHelpers.ts:7-11` (`getDaysOverdue`) floors millisecond differences; `utils/annualReportsUtils.ts:5-10` (`daysUntil`) uses date-fns calendar days and a terminal-status set; `utils/panelHelpers.ts:47-65` uses `Math.ceil` and ignores terminal status. `constants/sharedConstants.ts:1-7` separately owns calendar-date parsing.
- **Behavioral differences:** floor versus calendar versus ceil semantics; only one path excludes terminal reports. Consumers are `OverdueBanner`, `SeasonReportsTable.tsx:22-41,119-123`, and `ReportAlertBanners.tsx:55`.
- **Likely source of truth:** backend lifecycle state plus a clearly documented Israel/UTC day-distance contract.
- **Consolidation recommendation:** add canonical `is_overdue` and day-distance fields to annual list/detail responses, then remove local variants. If backend work is deferred, keep one timezone-defined helper and terminal set.
- **Is consolidation worth doing?** Yes. It fixes contradictory banners and prevents boundary drift. Add terminal and local-midnight tests.

## DU-04 — annual-report status labels and season stages have drifted

**Severity/confidence:** P2 · Confirmed; canceled-stage omission also contributes to `CT-08`.

- **Canonical rule or behavior:** the likely frontend owner is `frontend/src/features/annualReports/api/utils.ts:7-17`, `STATUS_LABELS`.
- **Duplicated implementations:** the same file defines `SEASON_PROGRESS_STAGES` at `:74-81`; `frontend/src/features/timeline/labels.ts:28-36,57` defines `ANNUAL_REPORT_STATUS_LABEL_MAP`. The timeline file already imports the annual-report map for audit value labels at `:38-49`.
- **Behavioral differences:** season pending copy is shortened and canceled is absent; timeline renders `pending_client` and `closed` with different Hebrew wording than annual-report screens.
- **Likely source of truth:** one typed annual-report presentation map; stage order is a distinct presentation concern derived from it.
- **Consolidation recommendation:** derive stages/options from the typed map while explicitly ordering and filtering; use context-specific copy only when named as such.
- **Is consolidation worth doing?** Yes, small and low risk because a real stage omission already exists. Add a completeness test against `AnnualReportStatus`.

## DU-05 — annual-report client-type labels/options have three owners and two getter contracts

**Severity/confidence:** P2 · Confirmed.

- **Canonical rule or behavior:** a client type should map to one short display label; form options may deliberately add “form” wording.
- **Duplicated implementations:** `frontend/src/features/annualReports/api/utils.ts:38-48` (`clientTypeLabels`); `constants/panelConstants.ts:16-24` (`CLIENT_TYPE_LABELS`); `utils/panelHelpers.ts:26-27` (a second `getClientTypeLabel`, accepting a report rather than a type); and `constants/sharedConstants.ts:11-19` (`CLIENT_TYPE_OPTIONS`).
- **Behavioral differences:** the short maps are effectively copies; options contain longer context-specific copy; identical getter names hide different input shapes.
- **Likely source of truth:** one annual-report client-type presentation module.
- **Consolidation recommendation:** own the short map/getter once; derive options or name long-copy variants explicitly. Do not force every context to use identical prose.
- **Is consolidation worth doing?** Yes, modest P2 cleanup. Low risk; add map/option completeness tests rather than component snapshots.

## DU-06 — VAT reporting-frequency type and labels are duplicated across client and VAT features

**Severity/confidence:** P2 · Confirmed.

- **Canonical rule or behavior:** monthly/bimonthly VAT reporting frequency is a cross-domain tax concept, not private client UI state.
- **Duplicated implementations:** `frontend/src/features/clients/constants.ts:106-127` defines `VAT_TYPES`, labels, and options; `frontend/src/features/vatReports/constants/vatConstants.ts:53-62` defines `VAT_PERIOD_TYPE_OPTIONS` and `VAT_PERIOD_TYPES`.
- **Behavioral differences:** Hebrew labels differ only by ASCII versus Hebrew maqaf. `vatReports/api/contracts.ts:1-5` already imports `VatType` from the clients feature's private API, demonstrating unclear ownership.
- **Likely source of truth:** a small shared tax/reporting-frequency module, or a clearly public domain owner.
- **Consolidation recommendation:** move the type/map once and derive both option arrays. Avoid importing a feature's private API contract from another feature.
- **Is consolidation worth doing?** Yes, but after correctness work. Low risk and useful boundary cleanup.

## DU-07 — charge create/edit forms duplicate the same core field block

**Severity/confidence:** P2 · Confirmed.

- **Canonical rule or behavior:** amount, type, business, period, and months-covered are the domain core shared by charge create and draft edit.
- **Duplicated implementations:** `frontend/src/features/charges/components/form/ChargesCreateModal.tsx:88-225` and `ChargeEditModal.tsx:51-173` repeat options and field rendering/wiring.
- **Behavioral differences:** create additionally owns client selection and initial-business handling; edit owns draft defaults and description. The duplicated core itself is stable. Validation/payload logic already shares `chargeCoreSchema` and `toChargeCorePayload` in `features/charges/schemas.ts:6-40,54-70`.
- **Likely source of truth:** the existing charge schema plus a domain-specific field component.
- **Consolidation recommendation:** extract `ChargeCoreFields` with normal field props/form context. Keep create/edit modal orchestration separate; do not introduce a generic configurable form builder.
- **Is consolidation worth doing?** Yes. This is the largest UI field clone with meaningful change risk. Low-medium migration risk; add both create/edit interaction tests.

## DU-08 — global and client-scoped charge workspaces repeat page composition

**Severity/confidence:** P2 · Confirmed.

- **Canonical rule or behavior:** `useChargesPage` already owns pinned versus global data/filter behavior.
- **Duplicated implementations:** `frontend/src/features/charges/pages/ChargesPage.tsx:10-53` and `components/shared/ClientChargesTab.tsx:17-65` both wire the create action, `ChargesStatsSection`, `FilterPanel`, `ChargesTableBlock`, `ChargesCreateModal`, and `SendNotificationModal` from the same hook result.
- **Behavioral differences:** the global page uses `PageHeader`/`PageStateGuard`; the client view uses `DetailTabPanel`, passes loading/error to the table, and changes subtitle/client columns. Those shells are legitimate; the repeated workspace body and large table prop mapping are not.
- **Likely source of truth:** a charge-domain `ChargesWorkspaceBody` accepting the already prepared hook model and shell-specific flags/slots.
- **Consolidation recommendation:** share the body/table/modal composition, leaving each page shell and title/breadcrumb ownership local. Resolve `CT-13` capability naming before extracting so duplication is not frozen into a misleading `isAdvisor` API.
- **Is consolidation worth doing?** Yes, after permission correction. Low-medium UI risk; test both global and pinned-client variants.

## DU-09 — global and client-scoped binder workspaces repeat the complete table/drawer stack

**Severity/confidence:** P2 · Confirmed.

- **Canonical rule or behavior:** `useBindersPage` in `frontend/src/features/binders/hooks/useBindersPage.ts:52-299` already switches client picker/columns/query scope using `pinnedClient` and pre-wires dialogs/drawers.
- **Duplicated implementations:** `pages/BindersPage.tsx:13-63` and `components/shared/ClientBindersTab.tsx:19-72` both render intake action, stats, filters, `PaginatedDataTable`, `BindersPageDialogs`, `BinderDetailDrawer`, and receive `DetailDrawer`/`BinderReceivePanel`.
- **Behavioral differences:** page versus tab shell, client-specific empty copy, and tab loading/fetching props. Those differences fit explicit props/slots.
- **Likely source of truth:** a binder-domain workspace body consuming the existing controller model.
- **Consolidation recommendation:** extract the repeated stats/filter/table/dialog/drawer body; leave `PageHeader`/`PageStateGuard` and `DetailTabPanel` outside. Do not add another controller hook.
- **Is consolidation worth doing?** Yes. It prevents the two operational surfaces from drifting and removes a large JSX clone; medium visual risk, so cover loading/error/empty/deep-link/receive states.

## DU-10 — notification trigger labels/domains are duplicated despite server-enriched responses

**Severity/confidence:** P2 · Confirmed · same contract issue as `CT-29`.

- **Canonical rule or behavior:** `backend/app/notifications/models/notification.py:55-85` owns `TRIGGER_LABELS` and `TRIGGER_DOMAIN`; `backend/app/notifications/services/notification_service.py:33-70` already adds `trigger_label` and `domain_label` to list/detail DTOs.
- **Duplicated implementations:** `frontend/src/features/notifications/api/contracts.ts:3-43` repeats all trigger values and labels; `constants.ts:11-24,42-48,55-62` repeats domains/options; `SendNotificationModal.tsx:17-24` builds chooser labels; `NotificationsColumns.tsx:18-26` retains frontend fallback maps.
- **Behavioral differences:** the VAT label uses ASCII quotes (`מע\"מ`) in the frontend and Hebrew gershayim (`מע״מ`) in the backend; future backend labels/domains can change without updating compose options.
- **Likely source of truth:** backend notification metadata. Read DTOs are already authoritative, but create/preview needs allowed-trigger metadata.
- **Consolidation recommendation:** expose manual trigger option metadata (value, label, domain, allowed context/role) from the backend and use response labels on reads. Keep only a generic unknown-value fallback locally.
- **Is consolidation worth doing?** Yes, medium value and low UI risk once metadata exists. Add API contract/completeness tests.

## Acceptable duplication — no extraction recommended

- **Annual income versus expense forms:** they already share `FinancialAddFormShell` and `FinancialAmountDescriptionFields`; remaining branches express genuinely different domain fields and payloads. A config-driven form builder would obscure them.
- **Client versus business status labels:** identical current values belong to distinct backend enums that can evolve independently.
- **Feature stats sections:** `ChargesStatsSection` and `VatWorkItemsStatsSection` repeat simple composition over shared stat primitives. The local code is clearer than a generic stats configuration layer.
- **Short listbox internals:** `DatePickerInlineSelect` and `SelectDropdown` share a small pattern but have different keyboard/selection behavior. Wait for a third stable use before extracting.
- **Tiny formatting/conditional pairs:** two-line uses of standard currency/date/status primitives were inspected and intentionally not abstracted merely to reduce line count.

## Consolidation order

1. Delete the incorrect tax-credit copy (`DU-01`).
2. Add backend metadata and remove VAT deduction-law duplication (`DU-02`).
3. Correct the visible terminal-state defect, then consolidate deadline semantics (`DU-03`).
4. Extract charge core fields (`DU-07`).
5. Resolve permissions, then share charge and binder workspace bodies (`DU-08`, `DU-09`).
6. Consolidate the small typed maps (`DU-04`–`DU-06`) and notification metadata (`DU-10`).
