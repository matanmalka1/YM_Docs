# Frontend code-health audit — frontend/backend contract leaks

Audit date: 2026-07-22
Scope: handwritten frontend DTOs, serializers, calculations, permission gates, aggregate derivations, route construction, response handling, and related backend schemas/services/tests. Production code was not modified.

## Summary

| Severity | Findings |
|---|---:|
| P0 | 17 |
| P1 | 4 |
| P2 | 5 |
| P3 | 3 |
| **Total** | **29** |

Generated OpenAPI and enum-sync checks pass. The failures below sit above that generated layer: handwritten types, payload builders, UI calculations, capability gates, route strings, and response fields that are discarded or reconstructed. “No frontend test” means no focused test of the named behavior was found after checking colocated and feature tests; related backend coverage is identified separately.

## CT-01 — VAT edit serializer violates PATCH presence/null semantics

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/vatReports/schemas/invoice.schema.ts:20-30,54,62-75`, symbols `invoiceCommonFields`, `vatInvoiceEditSchema`, `buildInvoicePayloadBase`, and `toInvoiceEditPayload`, serialize create and edit values through one builder. It forces `expense_category` and `document_type` to `null` at `:64,66`, but turns a cleared `counterparty_id` into `undefined` at `:70`.
- **All known consumers:** `frontend/src/features/vatReports/components/list/VatInvoiceEditRow.tsx:39-54` is the only edit serializer consumer. `VatInvoiceTable.tsx:243-251` renders it for both income and expense `VatInvoiceTab.tsx:50-54`. Edit defaults omit `document_type` and `rate_type` (`VatInvoiceEditRow.tsx:39-49`) and reconstruct gross from net + VAT at `:42` instead of using response `gross_amount`.
- **Backend contract/proof:** `backend/app/vat/schemas/vat_invoice_update.py:26-60` rejects explicit null for non-clearable fields including `expense_category`, while counterparty fields and `document_type` are clearable. `backend/app/vat/api/vat_routes_data_entry.py:81-107` uses `model_dump(exclude_unset=True)`. `backend/app/vat/services/vat_data_entry_invoice_update_service.py:49-58,105-135` documents presence as intent, validates the merged ID pair, and applies only sent keys.
- **Problem:** every income edit sends `expense_category: null` and receives 422; an expense edit sends `document_type: null` and can silently erase classification; the UI cannot clear a nullable counterparty ID because it omits the key.
- **Recommendation/risk:** split create/edit serializers, make edit output presence-aware, seed/render or omit document/rate fields, and use response `gross_amount`. Medium migration risk because persisted fields change.
- **Tests:** no frontend serializer/edit-row tests. Backend null-clearing and counterparty-pair behavior is covered by `backend/tests/vat/api/test_vat_reports_invoices_update_and_filters.py`.

## CT-02 — OSEK-PATUR ceiling warning is discarded

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `VatInvoiceResponse` in `frontend/src/features/vatReports/api/contracts.ts:166-186` omits backend `ceiling_warning`. `vatReportsApi.addInvoice` (`api/vatReports.api.ts:103-105`) returns that incomplete type. `runMutationWithFeedback` in `hooks/useVatInvoiceMutations.ts:16-29` collapses a result to `boolean`; `useAddInvoice` (`:31-46`) and `components/form/VatInvoiceAddForm.tsx:43-45` therefore cannot inspect the warning.
- **All known consumers:** the add mutation/form is mounted from both income and expense VAT invoice tabs; no other consumer preserves the create response.
- **Backend contract/proof:** `backend/app/vat/schemas/vat_invoice_schema.py:109-142` includes `ceiling_warning`; `backend/app/vat/api/vat_routes_data_entry.py:26-59` populates it. `docs/domains/vat.md:113` defines the 80% threshold as a non-blocking warning.
- **Recommendation/risk:** include the response field, return the mutation result, and show a non-blocking warning after create. Low-medium UI risk.
- **Tests:** no frontend warning test. Threshold behavior is covered by `backend/tests/vat/service/test_data_entry_service_additional.py:56-85`.

## CT-03 — annual expense creation overrides backend statutory recognition defaults

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `DEFAULT_RECOGNITION_RATE = '100'` in `frontend/src/features/annualReports/constants/financialConstants.ts:3`; `useExpenseLineForm` in `hooks/useFinancialLineForm.ts:47-88` initializes/resets every new expense to 100 and always passes it to `buildExpensePayload`; `utils/financialHelpers.ts:67-91` always emits `recognition_rate: String(rate / 100)`.
- **All known consumers:** expense create/edit in `IncomeExpenseTab`; every new vehicle, communication, and general expense uses this form/builder.
- **Backend contract/proof:** `backend/app/annual_reports/domain/expense_rules.py:5-16` owns defaults of vehicle 0.75, communication 0.80, other 1.00. `backend/app/annual_reports/services/annual_report_financial_line_service.py:196-230` applies the category default only when `recognition_rate` is omitted (`:217-220`).
- **Problem:** the frontend always sends 1.0 and bypasses statutory server defaults, changing recognized expenses and taxable income.
- **Recommendation/risk:** omit an untouched recognition rate on create; retain explicit edit override semantics or expose server metadata for previews. Medium behavior/migration risk.
- **Tests:** no frontend builder/default tests. Backend expense-rule/financial-line service tests cover the defaulting behavior.

## CT-04 — recognition ratios are rendered as raw percentages

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/annualReports/components/financials/FinancialLineRow.tsx:46-49` compares a response ratio with 100 and interpolates it directly with `%`; `components/tax/DeductionsTab.tsx:50-63` repeats the same assumption.
- **All known consumers:** `IncomeExpenseTab.tsx:93-107` passes raw `recognition_rate` to `FinancialLineRow`; `DeductionsTab` passes it to its recognition display.
- **Backend contract/proof:** `ExpenseLineResponse.recognition_rate` is a 0..1 ratio in backend/OpenAPI; backend defaults return 0.75, 0.80, or 1.00.
- **Problem:** 0.75 is displayed as `0.75%` and 1.0 as `1%`, not 75% and 100%.
- **Recommendation/risk:** convert ratio to percent once at the display boundary. Low code risk; visible output changes.
- **Tests:** no focused frontend display tests. Add 0, 0.75, 0.8, and 1.0 cases.

## CT-05 — `TaxCreditsPanel` reimplements tax calculations with wrong rules

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/annualReports/constants/taxConstants.ts:17-25` hardcodes point values and a 4.5% pension rate. `utils/taxHelpers.ts:27,48-99`, symbols `getCreditPointValue`, `buildCreditRows`, and `sumCreditRows`, calculate credits locally, label pension contribution as a training fund, turn it into a 4.5% credit, and omit donation credit. `components/tax/TaxCreditsPanel.tsx:12-45` displays the result.
- **All known consumers:** `TaxCreditsPanel` is rendered by `DeductionsTab.tsx:76`. `TaxCalculationTab.tsx:47-66,90-121` independently renders the server result, creating two conflicting annual-report views.
- **Backend contract/proof:** authoritative point values are 2024=2904 (`constants_2024.py:111-119`), 2025=2820 (`constants_2025.py:173-181`), 2026=2904 (`constants_2026.py:185-194`) under `backend/tax_rules_config/app/tax_rules/financials/`. `backend/app/annual_reports/annual_report_tax_engine.py:54-105` treats pension deduction directly and computes point value, donation credit, and total; `annual_report_tax_service.py:94-96` passes the stored pension amount directly.
- **Recommendation/risk:** delete local tax arithmetic and render backend fields. Add a backend breakdown DTO if labeled rows are required. Medium risk/backend coordination.
- **Tests:** no frontend `buildCreditRows` tests. Backend tax-engine tests cover yearly constants and rule output.

## CT-06 — merged advance-payment batches retain stale paid/not-paid counts

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `mergeAdvancePaymentBatches` in `frontend/src/features/advancedPayments/utils/advancePaymentUtils.ts:16-54` merges same-due-date groups and sums client/pending/overdue/missing/due/amount fields (`:33-45`) but not `paid_count` or `not_paid_count`; spread retains the first group's values.
- **All known consumers:** `hooks/useAdvancePaymentsPage.ts:42-45` always applies the merger to display batches. `components/table/AdvancePaymentBatchRow.tsx:19-50` renders merged client count beside stale paid/not-paid counts.
- **Backend contract/proof:** `backend/app/advance_payments/services/advance_payment_analytics_service.py:158-193` returns per-period counts; `repositories/advance_payment_batch_repository.py:43-122` can produce multiple period groups sharing one due date.
- **Recommendation/risk:** sum both fields and assert every additive aggregate; consider returning canonical due-date groups from the backend. Low change risk.
- **Tests:** `frontend/src/features/advancedPayments/utils/advancePaymentUtils.test.ts:24-53` claims to cover workflow totals but omits these assertions.

## CT-07 — mixed tax-calendar groups are labeled Done when any item is done

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `getTaxCalendarGroupStateLabel` and `getTaxCalendarGroupStateVariant` in `frontend/src/features/taxCalendar/helpers.ts:50-60` check `done_count > 0` before overdue/open state.
- **All known consumers:** only `frontend/src/features/taxCalendar/components/shared/ClientTaxCalendarList.tsx:14-21,58-91`.
- **Backend contract/proof:** `backend/app/tax_calendar/services/tax_calendar_grouped_service.py:111-140` computes counts; `_filter_groups_by_status` at `:145-158` defines done only when linked items exist and both open/overdue are zero, and overdue whenever `overdue_count > 0`.
- **Recommendation/risk:** mirror backend precedence (overdue first; done only when all linked work is done) or return aggregate state from the server. Low risk.
- **Tests:** `frontend/src/features/taxCalendar/helpers.test.ts:1-37` tests routes only; add mixed open/done/overdue cases.

## CT-08 — canceled annual reports are omitted and can be classified as in progress

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `buildStats` in `frontend/src/features/dashboard/hooks/useSeasonSummary.ts:14-31` computes `inProgress = total - not_started - done` without subtracting canceled. `SeasonInsightsCarousel.tsx:84-125` displays it. `SEASON_PROGRESS_STAGES` in `features/annualReports/api/utils.ts:74-81` omits canceled; `components/season/SeasonProgressBar.tsx:12-66` divides included stages by a total that can include canceled.
- **All known consumers:** dashboard season carousel and annual-report season progress bar.
- **Backend contract/proof:** `SeasonSummaryResponse` in `backend/app/annual_reports/schemas/annual_report_responses.py:137-149` includes `canceled`, but `services/annual_report_season_service.py:32-56` fails to populate it, leaving zero while total can include canceled rows.
- **Recommendation/risk:** populate canceled on the backend, subtract/show it in both frontend views, and define whether canceled belongs in completion percentage. Medium cross-layer risk.
- **Tests:** no cancellation-focused season summary/progress tests found.

## CT-09 — notification send workflow is hidden from secretaries

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** advisor-only gates occur in `frontend/src/features/notifications/hooks/useNotificationsPage.ts:34-36,87-91,150,182-191`, `frontend/src/features/notifications/pages/NotificationsPage.tsx:18-24,53`, `frontend/src/features/notifications/components/shared/NotificationsTab.tsx:21-39,61`, `frontend/src/features/notifications/components/drawer/NotificationDrawer.tsx:24,42-63,76-83`, and `frontend/src/features/notifications/components/list/NotificationsColumns.tsx:29-42,103-109`. `frontend/src/features/vatReports/components/detail/VatWorkItemHeaderActions.tsx:18-35` returns early for non-advisors, hiding reminders with advisor-only export.
- **All known consumers:** global notifications page, client notification tab/drawer/list actions, and VAT work-item header reminder action.
- **Backend contract/proof:** `backend/app/notifications/api/notification_routes.py:37-41,98-138` permits Advisor and Secretary for preview/send. `docs/domains/notifications.md:165` records decision F-024 with no advisor-only guard.
- **Recommendation/risk:** add semantic `canSendNotifications` for both roles and keep export permission separate. Low-medium risk.
- **Tests:** no frontend notification role-matrix tests found.

## CT-10 — signature request management is hidden from secretaries

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/hooks/useRole.ts:4-20` defines `can.editClients` as advisor-only. `frontend/src/features/clients/components/detail/ClientDetailsOverviewTab.tsx:95` passes it as `canManage`; `frontend/src/features/signatureRequests/components/shared/SignatureRequestsCard.tsx:21-26,47-55,68-96,126-138` hides create/manage; `frontend/src/features/signatureRequests/components/list/SignatureRequestRowActions.tsx:33-36,75-108` hides row actions.
- **All known consumers:** client signature card creation, cancel/reminder/audit row actions.
- **Backend contract/proof:** `backend/app/signature_requests/api/signature_request_routes_client.py:25-90` permits both roles for list/cancel; `signature_request_routes_advisor.py:27-96`, despite its name, permits both for create/pending.
- **Recommendation/risk:** define a signature-specific capability for both roles instead of reusing client-edit permission. Low risk.
- **Tests:** no frontend permission coverage found.

## CT-11 — VAT work-item creation is hidden from secretaries

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/vatReports/pages/VatWorkItemsPage.tsx:21-32` shows create only to advisors and presents a secretary view-only notice. `hooks/useVatWorkItemsPage.ts:58-63,201-210,318-321` repeats the gate and rejects secretary submission.
- **All known consumers:** global VAT work-items page and its create flow.
- **Backend contract/proof:** `backend/app/vat/api/vat_routes_intake.py:22-49` permits both roles and explicitly documents Advisor/Secretary access at `:34-38`.
- **Recommendation/risk:** expose `canCreateVatWorkItem` for both roles and test the role matrix. Low-medium risk.
- **Tests:** no focused frontend permission tests found.

## CT-12 — one VAT mutation flag over-grants invoice deletion

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `canMutateVatInvoices` in `frontend/src/features/vatReports/utils/vatHelpers.ts:8-23` equates every invoice mutation with backend `add_invoice`. `VatInvoiceTab.tsx:15-19,50-61` passes one `canEdit` flag to add/table UI. `VatInvoiceTable.tsx:206-230,254-267` exposes edit and delete together.
- **All known consumers:** income and expense invoice tables/add forms.
- **Backend contract/proof:** add/update allow both roles in `backend/app/vat/api/vat_routes_data_entry.py:26-30,81-84`; delete is advisor-only at `:110-114`. `backend/app/actions/services/vat_report_actions.py:31-41` gives secretaries only `add_invoice`; `backend/tests/actions/test_vat_report_actions.py:20-25` asserts it.
- **Recommendation/risk:** split add/edit/delete capabilities and gate deletion on advisor/server action. Medium authorization-UI risk.
- **Tests:** `frontend/src/features/vatReports/utils/vatHelpers.test.ts:48-55` covers only the broad add flag and misses deletion.

## CT-13 — charge permissions contradict backend routes, tests, and action descriptors

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/charges/hooks/useChargesPage.ts:69-76` allows both roles inside action hooks, but deep-link create (`:92-99`), submit (`:131-143`), columns (`:102-129`), and returned permissions (`:209-211`) use advisor-only UI gates. `pages/ChargesPage.tsx:13-25`, `components/shared/ClientChargesTab.tsx:22-38`, `components/list/ChargesTableBlock.tsx:42-65`, and `ChargeColumns.tsx:101-145` hide create/bulk/row actions and label secretaries view-only.
- **All known consumers:** global charges page and pinned client charges tab; detail uses `useChargeDetailsPage.ts:19,67-104`, which misleadingly returns `isAdvisor: isAdvisor || isSecretary` but still receives empty secretary action descriptors.
- **Backend contract/proof:** create/update/lifecycle/bulk/delete all permit both roles in `backend/app/charges/api/charge_routes.py:41-59,62-122,168-202`; `docs/domains/charges.md:17-31` says the same; `backend/tests/charges/api/test_charges_api_authorization.py:40-64` proves secretary create/issue/cancel. Contradictorily, `backend/app/actions/services/charge_actions.py:29-34` returns no descriptors to secretaries and `backend/tests/actions/test_charge_actions.py:31-34` locks that behavior.
- **Recommendation/risk:** resolve backend policy first, then emit semantic action capabilities and remove role-name gates. Secretary create is unambiguously blocked by frontend-only checks; other actions require coordinated alignment. Medium risk.
- **Tests:** backend tests currently prove both sides of the contradiction; no frontend role-matrix tests.

## CT-14 — terminal annual reports can show overdue and completed alerts together

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `getAlertBanners` in `frontend/src/features/annualReports/utils/panelHelpers.ts:34-93` derives overdue from date alone (`:47-65`) and then adds submitted/closed success/info at `:69-82`.
- **All known consumers:** only `components/panel/ReportAlertBanners.tsx:55`.
- **Backend contract/proof:** `backend/app/annual_reports/repositories/annual_report_report_lifecycle_repository.py:38-54` limits overdue to open statuses (`:40-48`).
- **Recommendation/risk:** consume backend overdue metadata or use one helper with terminal semantics. Low risk.
- **Tests:** no terminal-past-deadline frontend tests found.

## CT-15 — frontend VAT deduction table duplicates and differs from backend tax rules

**Severity/confidence:** P1 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/vatReports/constants/vatConstants.ts:64-147` owns categories, labels, and `DEDUCTION_RATES`; `utils/viewHelpers.ts:31-38` emits fixed deduction hints.
- **All known consumers:** `components/form/VatInvoiceAddForm.tsx:37-41,167-171` is the sole hint consumer.
- **Backend contract/proof:** authoritative configuration is `backend/tax_rules_config/app/tax_rules/vat_deduction.py:5-180`, consumed at runtime by `backend/app/vat/vat_data_entry_common.py:92-99`. Rate precision, labels, and vehicle-condition semantics differ.
- **Recommendation/risk:** return category/rate/condition metadata or a computed preview from backend; remove frontend tax-law constants. Medium/backend-coordination risk.
- **Tests:** backend config tests cover rates; no frontend hint tests.

## CT-16 — annual deadline state is calculated three incompatible ways

**Severity/confidence:** P1 · High confidence; terminal divergence confirmed, midnight/DST impact needs runtime verification.

- **Current responsibility and evidence:** `frontend/src/features/annualReports/utils/sharedHelpers.ts:7-11` floors milliseconds; `utils/annualReportsUtils.ts:5-10` uses date-fns calendar days and terminal statuses; `utils/panelHelpers.ts:47-65` uses ceil milliseconds and no terminal check. Parsing lives in `constants/sharedConstants.ts:1-7`.
- **All known consumers:** `OverdueBanner`, `SeasonReportsTable.tsx:22-41,119-123`, and `ReportAlertBanners` respectively.
- **Backend contract/proof:** `annual_report_report_lifecycle_repository.py:38-54` uses server UTC/open statuses. Some report DTOs already expose `days_until_deadline`; annual list/detail do not.
- **Recommendation/risk:** add canonical `is_overdue`/day-distance fields to annual list/detail; otherwise use one timezone-defined helper. Medium boundary risk.
- **Tests:** no focused frontend deadline tests.

## CT-17 — status/deadline API wrapper promises a fuller response than the endpoint returns

**Severity/confidence:** P2 · Confirmed

- **Current responsibility and evidence:** `transitionStatus` and `updateDeadline` in `frontend/src/features/annualReports/api/annualReports.status.api.ts:5-9,24-30` promise `AnnualReportFull`.
- **All known consumers:** `hooks/useReportMutations.ts:18-50` ignores the returned value and invalidates queries; `updateDeadline` has no live frontend consumer found.
- **Backend contract/proof:** `backend/app/annual_reports/api/annual_report_routes_status.py:30-60,92-122` returns base `AnnualReportResponse`; only submit at `:63-89` returns detail/full. OpenAPI agrees.
- **Recommendation/risk:** type the exact base response or `void` when invalidation is the intended contract. Low risk.
- **Tests:** no frontend response-shape tests.

## CT-18 — frontend derives `id_number_type` that the client-create backend owns

**Severity/confidence:** P2 · Confirmed

- **Current responsibility and evidence:** `deriveCreateClientIdNumberType` in `frontend/src/features/clients/constants.ts:221-222` is used only by `utils/createClientFormUtils.ts:29-51`, which sends it at `:35`.
- **All known consumers:** client create form payload mapping only.
- **Backend contract/proof:** `backend/app/clients/client_create_policy.py:9-14` derives it; `backend/app/clients/schemas/client_requests.py:28,51-60` makes it optional and validates consistency; `docs/domains/clients.md:184,189` identifies backend derivation.
- **Recommendation/risk:** omit the field and delete the frontend derivation. Low risk; current values match.
- **Tests:** no frontend mapper test found; enum sync passes.

## CT-19 — user-audit DTO drops actor and target display names

**Severity/confidence:** P2 · Confirmed

- **Current responsibility and evidence:** `UserAuditLogResponse` in `frontend/src/features/users/api/contracts.ts:16-26` omits `actor_display_name` and `target_display_name`; `usersApi.listAuditLogs` at `api/users.api.ts:53-57` returns that type.
- **All known consumers:** `features/users/components/AuditLogsDrawer.tsx:87-103`, which can show action/email/reason/date but not who acted or the target.
- **Backend contract/proof:** `backend/app/users/schemas/user_management.py:60-73` exposes both fields; `backend/app/users/services/user_audit_log_service.py:75-93` maps both persisted values into the response.
- **Recommendation/risk:** complete the DTO and render attribution with null fallback. Low risk.
- **Tests:** no frontend audit-log response/render tests.

## CT-20 — tax-calendar group item omits backend `id_number`

**Severity/confidence:** P3 · Confirmed

- **Current responsibility and evidence:** `TaxCalendarGroupItem` in `frontend/src/features/taxCalendar/api/taxCalendar.api.ts:52-66` omits `id_number`. Backend `TaxCalendarGroupItem` declares it at `backend/app/tax_calendar/schemas/tax_calendar_grouped.py:23-38`, and `tax_calendar_grouped_items_service.py:41-52` populates it from the legal entity.
- **All known consumers:** tax-calendar group/list components; none currently reads `id_number`.
- **Problem/recommendation/risk:** harmless current drift, but handwritten DTOs no longer mirror the server. Align when touching the contract or deliberately document the thin DTO. Negligible risk.
- **Tests:** no field-shape frontend test.

## CT-21 — annual annex DTO omits `data_version`, whose live purpose is unclear

**Severity/confidence:** P3 · High confidence

- **Current responsibility and evidence:** `AnnexDataLine` in `frontend/src/features/annualReports/api/contracts.ts:301-310` omits `data_version` from `backend/app/annual_reports/schemas/annual_report_annex.py:10-19`.
- **All known consumers:** annual annex data panel/table hooks; no consumer attempts concurrency/version checks.
- **Backend contract/proof:** the response exposes the field, but current repository update logic at `backend/app/annual_reports/repositories/annual_report_annex_data_repository.py:79-89` does not increment/use it.
- **Recommendation/risk:** decide whether optimistic concurrency is intended; expose and enforce it, or remove the stale backend field. No frontend-only fix recommended. Low current risk; runtime/product intent unresolved.
- **Tests:** no version-behavior tests found.

## CT-22 — charge create type permits fields the endpoint silently ignores

**Severity/confidence:** P3 · Confirmed

- **Current responsibility and evidence:** `CreateChargePayload` in `frontend/src/features/charges/api/contracts.ts:78-87` permits `description` and `annual_report_id`.
- **All known consumers:** charge create mutation/modal; current `toCreateChargePayload` in `frontend/src/features/charges/schemas.ts:54-70` emits neither unsupported field.
- **Backend contract/proof:** `ChargeCreateRequest` in `backend/app/charges/schemas/charge.py:12-18` accepts neither; default Pydantic extra handling ignores them.
- **Recommendation/risk:** narrow create payload to prevent a future caller believing the fields persisted; description remains PATCH-only. Negligible current risk.
- **Tests:** current charge mapper/cache tests do not exercise unsupported extra keys.

## CT-23 — Attention Board “show all” link targets an unreachable route

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `AttentionBoard` in `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:125-169` renders the overflow action at `:142-150` with `to="/work"` (`:145`).
- **All known consumers:** dashboard advisor path in `frontend/src/features/dashboard/pages/DashboardPage.tsx:102,122`, via the feature barrel `features/dashboard/index.ts:1`. The link appears only when `displayTotal > items.length`.
- **Frontend route proof:** `frontend/src/router/AppRoutes.tsx:188` registers `/work-queue`; no `/work` route exists. The wildcard at `:201` redirects `/work` to `/`, so the action cannot reach the queue.
- **Recommendation/risk:** use a canonical `WORK_QUEUE_ROUTE`/route builder and add a rendered overflow-link route test. Very low code risk; behavior correction.
- **Tests:** no `AttentionBoard` component/link test found.

## CT-24 — signature-create modal accepts `signerPhone` but ignores it

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `CreateSignatureRequestModal` declares `signerPhone?: string` at `frontend/src/features/signatureRequests/components/form/CreateSignatureRequestModal.tsx:21-31`, but destructuring/state/UI/payload at `:33-82,149-166` never reads it and never sends `signer_phone`.
- **All known consumers:** `components/shared/SignatureRequestsCard.tsx:126-138` supplies `client.phone` at `:131` and is the only caller supplying the prop; `SignatureRequestsDashboardPanel.tsx:208-216` uses the modal without a preselected client/phone.
- **Backend contract/proof:** frontend `CreateSignatureRequestPayload` supports `signer_phone` (`api/contracts.ts:79-92`). Backend `SignatureRequestCreateRequest` accepts it (`backend/app/signature_requests/schemas/signature_request.py:82-94`), route forwards it (`api/signature_request_routes_advisor.py:40-64`), and service persists/falls back it (`services/signature_request_creation_service.py:40-52,73-79,99-108`).
- **Recommendation/risk:** initialize an editable phone field from `signerPhone` and include trimmed `signer_phone`, or remove the prop only if product explicitly relies on server contact fallback. Low-medium privacy/contact-data risk.
- **Tests:** no modal payload test. Backend fallback behavior is covered by `backend/tests/signature_requests/service/test_signature_requests.py:270-296`; direct frontend-provided phone is not covered end to end.

## CT-25 — signature-create modal validates only nonempty values while backend enforces lengths

**Severity/confidence:** P0 · Confirmed

- **Current responsibility and evidence:** `CreateSignatureRequestModal.handleSubmit` at `frontend/src/features/signatureRequests/components/form/CreateSignatureRequestModal.tsx:70-84` checks only nonempty title/client/signer; inputs at `:135-166` have `required`/email type but no min/max constraints or schema. One-character title/name and >2000-character description can be submitted.
- **All known consumers:** both `SignatureRequestsCard.tsx:126-138` and `SignatureRequestsDashboardPanel.tsx:208-216` use this modal.
- **Backend contract/proof:** `backend/app/signature_requests/schemas/signature_request.py:82-94` requires title 3..200, description <=2000, signer name 2..100, positive IDs, and expiry 1..90.
- **Recommendation/risk:** add a typed Zod/react-hook-form schema matching the request, inline errors, and exact payload tests; keep the backend validation authoritative. Low-medium UI change risk and removes routine 422s.
- **Tests:** no frontend modal validation tests; backend API tests cover successful creates but no focused boundary matrix was found.

## CT-26 — tax-calendar settings parses English warning prose as an API protocol

**Severity/confidence:** P1 · High confidence

- **Current responsibility and evidence:** `translateWarning` in `frontend/src/features/taxCalendarSettings/pages/TaxCalendarSettingsPage.tsx:62-77` regex-parses two exact English sentence formats. Unknown text is returned raw at `:76`; `warnings` maps it at `:183-188` and renders it in Hebrew admin UI at `:277-283`.
- **All known consumers:** tax-calendar settings summary warning list only. Bootstrap warnings at `:285-294` are not translated at all; only their count changes alert tone.
- **Backend contract/proof:** `backend/app/tax_calendar/services/tax_calendar_settings_calendar_service.py:60-87` constructs English strings with punctuation/wording embedded, then returns raw `warnings` at `:91-97,100-117`. Backend tests such as `backend/tests/tax_calendar/service/test_settings_calendar_service.py:62-85,106-113` assert substrings/free-form warning behavior, not a structured schema.
- **Recommendation/risk:** return structured warnings `{code, year, obligation_type, expected, found}` and localize by code; retain raw text only as diagnostics. Medium backend/API migration risk.
- **Tests:** no frontend `translateWarning` tests. Runtime behavior for any changed backend wording is an untranslated English fallback.

## CT-27 — dashboard services hardcode frontend hrefs instead of canonical entity links

**Severity/confidence:** P1 · High confidence

- **Current responsibility and evidence:** `backend/app/dashboard/services/dashboard_attention_service.py:46-60,90-104` maps work-queue source types directly to frontend strings; charge/binder links lose entity IDs (`/charges`, `/binders`). `dashboard_recent_activity_service.py:166-175,236-253` owns a second route switch for audit entities and metadata-derived child links.
- **All known consumers:** frontend `AttentionBoard` follows response `item.href` at `frontend/src/features/dashboard/components/panels/AttentionBoard.tsx:72-78`; dashboard recent-activity components likewise consume backend `href` from `dashboard/api/contracts.ts`. No frontend route validation wraps these values.
- **Canonical-source proof:** `backend/app/common/entity_links.py:1-59` explicitly exists as the single owner of frontend deep links; `backend/app/common/source_types.py:26-38` adapts work-queue sources and documents legitimate advance-payment/task exceptions. Search already consumes `entity_route` (`backend/app/search/services/search_service.py:5,68,120`).
- **Problem:** dashboard services can drift independently from router/entity-link changes and currently provide less-specific charge/binder links than canonical helpers. This also leaks frontend topology into backend domain services instead of returning entity descriptors.
- **Recommendation/risk:** preferably return `{entity_type, entity_id, client_record_id}` and construct routes in one frontend route registry; minimally call `entity_route`/`source_route` from one backend adapter. Medium cross-layer risk.
- **Tests:** `backend/tests/dashboard/service/test_dashboard_attention_service.py:18-40` currently locks `/charges`; `test_recent_activity_service.py:62-92` checks VAT hrefs. Canonical source routes are separately tested in `backend/tests/work_queue/test_work_queue_service.py:248-253`, proving duplicated contracts.

## CT-28 — annual detail PATCH accepts `Partial<Response>` including server-only fields

**Severity/confidence:** P2 · Confirmed

- **Current responsibility and evidence:** `annualReportsApi.patchReportDetails` in `frontend/src/features/annualReports/api/annualReports.api.ts:80-83` accepts `Partial<ReportDetailResponse>`. That response type at `api/contracts.ts:158-168` includes server-owned `report_id`, `created_at`, and `updated_at`. `hooks/useReportMutations.ts:52-69,89` repeats the same broad input; `useTaxCalculationTab.ts:36-45` derives its mutation input from the API method.
- **All known consumers:** `AnnualReportDetailForm.tsx:10-47` currently sends only client approval/notes; `useTaxCalculationTab.ts:58-66` currently sends only pension/other credits. Current callers are safe, but the public type permits false writes.
- **Backend contract/proof:** `AnnualReportDetailUpdateRequest` in `backend/app/annual_reports/schemas/annual_report_detail.py:25-31` accepts only six mutable fields; `ReportDetailResponse` at `:9-22` adds report/timestamp fields. Route `api/annual_report_routes_detail.py:36-53` uses the narrow request.
- **Recommendation/risk:** define/generated `AnnualReportDetailUpdatePayload` from the request schema and use it through API/hooks/forms. Low risk; prevents future silent ignored extras.
- **Tests:** backend detail tests in `backend/tests/annual_reports/api/test_annual_report_detail.py:28-68` cover valid fields, not extra-key rejection; no frontend compile/runtime contract test.

## CT-29 — notification trigger labels/domains duplicate backend metadata and already differ

**Severity/confidence:** P2 · Confirmed

- **Current responsibility and evidence:** `frontend/src/features/notifications/api/contracts.ts:3-43` handwrites trigger values and `TRIGGER_LABELS`; `constants.ts:11-24,42-48,55-62` handwrites domain labels/options; `SendNotificationModal.tsx:17-24` uses local maps for chooser options; `NotificationsColumns.tsx:18-26` uses them as read fallbacks.
- **All known consumers:** notification compose/preview trigger selector, global/client list columns, filter options, and any caller importing `TRIGGER_LABELS` from `features/notifications/api/index.ts:17`.
- **Backend contract/proof:** `backend/app/notifications/models/notification.py:39-85` owns the enum, `TRIGGER_LABELS`, and `TRIGGER_DOMAIN`; `services/notification_service.py:33-70` already enriches list/detail responses with `trigger_label`/`domain_label`. The VAT label differs today: frontend uses ASCII quotes `מע\"מ` (`contracts.ts:35`), backend uses Hebrew gershayim `מע״מ` (`notification.py:61`).
- **Recommendation/risk:** use server labels for reads and expose allowed manual-trigger metadata for compose/filter options; retain only an unknown-value fallback. Low-medium API/UI risk.
- **Tests:** backend search/notification tests consume server labels, including `backend/tests/search/api/test_search_matches.py:212`; no frontend label parity/completeness test.

## Recommended contract-fix order

1. `CT-01`, `CT-03`–`CT-05`: fix persisted VAT/annual-tax correctness first.
2. `CT-23`–`CT-25`, `CT-06`, `CT-07`: small, independently testable correctness fixes.
3. `CT-09`–`CT-13`: establish and test semantic role capabilities; resolve the charge contradiction before broad UI changes.
4. `CT-02`, `CT-08`, `CT-14`: preserve server warnings and aggregate/terminal state.
5. `CT-15`, `CT-16`, `CT-26`, `CT-27`: add structured backend metadata/state/links and remove string/rule reconstruction.
6. `CT-17`–`CT-22`, `CT-28`, `CT-29`: narrow and complete handwritten DTOs/payload types.

## Static-verification limits

- Deadline behavior at local midnight/DST needs browser/runtime tests; terminal-status disagreement is statically confirmed.
- Historic expense overrides may have been intentional per row, but the new-create path is confirmed to suppress backend category defaults.
- Charge authorization is contradictory within backend source and tests; product intent is required beyond unblocking secretary create.
- Whether annex `data_version` is planned concurrency state cannot be inferred from current reads/writes.
- Dashboard `href` ownership is structurally duplicated and current differences are confirmed; the preferred entity-descriptor contract requires backend/frontend design coordination.
