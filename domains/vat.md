## Scope
This file owns only:
- Canonical current-state documentation for the vat domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# VAT

The VAT domain manages period-based VAT work items for a `ClientRecord`: material intake, invoice data entry, review, filing, audit history, client summaries, and VAT exports. The implemented aggregate is `VatWorkItem`, with `VatInvoice` rows as source documents. VAT audit writes now use the generic `EntityAuditLog` model: work-item lifecycle events are recorded on `vat_work_item`, and invoice events are recorded on `vat_invoice`.

Last verified against code + backend/openapi.json: 2026-07-30 (W4 amendment branch).

## Endpoints

All paths listed below exist in `backend/openapi.json`.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/vat/work-items` | Create a VAT work item for a client period |
| `PATCH` | `/api/v1/vat/work-items/{item_id}` | Update safe operational metadata only (`assigned_to`, `pending_materials_note`) |
| `DELETE` | `/api/v1/vat/work-items/{item_id}` | Soft-delete a non-filed mistaken obligation |
| `POST` | `/api/v1/vat/work-items/{item_id}/materials-complete` | Move pending material intake to material received |
| `POST` | `/api/v1/vat/work-items/{item_id}/invoices` | Add an income or expense invoice to a work item |
| `GET` | `/api/v1/vat/work-items/{item_id}/invoices` | List invoices for a work item, optionally by invoice type |
| `PATCH` | `/api/v1/vat/work-items/{item_id}/invoices/{invoice_id}` | Update an invoice on an editable work item |
| `DELETE` | `/api/v1/vat/work-items/{item_id}/invoices/{invoice_id}` | Delete an invoice from an editable work item |
| `POST` | `/api/v1/vat/work-items/{item_id}/ready-for-review` | Move data entry to review |
| `POST` | `/api/v1/vat/work-items/{item_id}/send-back` | Advisor sends a reviewed item back for correction |
| `POST` | `/api/v1/vat/work-items/{item_id}/file` | Advisor files the VAT return |
| `GET` | `/api/v1/vat/work-items/{item_id}/readiness` | Shared closing gate: `is_ready` + blocking `issues` (assignee, final amount) |
| `POST` | `/api/v1/vat/work-items/{item_id}/amend` | Advisor creates a new correction record from a submitted chain tip |
| `POST` | `/api/v1/vat/work-items/{item_id}/withdraw` | Advisor withdraws an unsubmitted correction and restores its original |
| `GET` | `/api/v1/vat/work-items/{item_id}/chain` | List the period's correction chain, oldest first, including withdrawn corrections |
| `GET` | `/api/v1/vat/work-items/groups` | List grouped work-item summaries; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/groups/{group_key}/items` | List items (thin `VatWorkItemListItem`) in a grouped due-date bucket; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/lookup` | Look up one item by `client_record_id` and `period` |
| `GET` | `/api/v1/vat/clients/{client_record_id}/period-options` | Return valid period options for a client |
| `GET` | `/api/v1/vat/work-items/status-summary` | Count work items by status; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/{item_id}` | Get one enriched work item (full `VatWorkItemResponse`) |
| `GET` | `/api/v1/vat/clients/{client_record_id}/work-items` | List one client's work items (thin `VatWorkItemListItem`); paginated (default `page_size=200`), filterable by `year`, `period`, `status`, `assigned_to`, `due_after`/`due_before` (date, vs `due_date_effective`). Ordered by `period desc, id desc` |
| `GET` | `/api/v1/vat/work-items` | List work items across clients (thin `VatWorkItemListItem`); supports exact `client_record_id` and legacy fuzzy `client_name` filters |
| `GET` | `/api/v1/vat/clients/{client_record_id}/summary` | Get client-level VAT period and annual aggregates |
| `GET` | `/api/v1/vat/clients/{client_record_id}/export` | Export a client's VAT data as Excel or PDF |

Router sources: `backend/app/vat/api/vat_routes_intake.py`, `backend/app/vat/api/vat_routes_work_items.py`, `backend/app/vat/api/vat_routes_data_entry.py`, `backend/app/vat/api/vat_routes_status.py`, `backend/app/vat/api/vat_routes_filing.py`, `backend/app/vat/api/vat_routes_grouped.py`, `backend/app/vat/api/vat_routes_queries.py`, `backend/app/vat/api/vat_routes_client_summary.py`.

VAT audit inspection is now served by the audit domain's generic endpoint, not by a VAT-local route:

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/audit/{entity_type}/{entity_id}` | Generic entity trail; VAT uses `vat_work_item` for work-item lifecycle and `vat_invoice` for invoice history |

VAT export OpenAPI documents both successful download media types, Excel and PDF, as binary file responses rather than `application/json` with an empty schema (`backend/app/vat/api/vat_routes_client_summary.py`, `backend/openapi.json`).

List/detail DTO split: the three work-item list endpoints return the thin `VatWorkItemListItem` — identity, period, status, `net_vat`/`final_vat_amount`/`is_overridden`, displayed deadline fields, `closed_at`, `updated_at`, `available_actions`, and `amends_id` so an amendment never receives a plain-delete control. Detail-only fields (raw `total_*` amounts, `override_justification`, `submission_method`/`submission_reference`, `closed_by`/`closed_by_name`, `assigned_to`/`assigned_to_name`, `statutory_deadline`, the full amendment-chain state, `client_status`, `pending_materials_note`, and the server-computed `breakdown`) are served only by `GET /vat/work-items/{item_id}` as the full `VatWorkItemResponse`; the list-row click navigates to the detail page, which refetches by id. The `breakdown` carries authoritative income net/output VAT totals plus expense rows grouped by category (`category`, server label, stored deduction rate, signed net amount, gross VAT, and deductible VAT) and the corresponding expense/input totals. The frontend renders this object and does not recompute tax amounts from invoice rows. (`backend/app/vat/schemas/vat_report.py`, `backend/app/vat/api/vat_serializers.py`, `backend/app/vat/repositories/vat_invoice_aggregation_repository.py`).

## Model & fields

**`VatWorkItem`** (`vat_work_items`) is the root VAT workflow record. Columns: `id`; `client_record_id` FK to `client_records.id` non-null; `created_by` FK to `users.id` non-null; `assigned_to` FK to `users.id` nullable; `period` `String(7)` non-null; `period_type` `VatType` non-null; `status` shared `ObligationStatus` non-null; `pending_materials_note` nullable; `total_output_vat`, `total_input_vat`, `net_vat`, `total_output_net`, `total_input_net` non-null `Numeric(12,2)` totals; filing fields `final_vat_amount`, `is_overridden`, `override_justification`, `submission_method`, `submission_reference`; closing facts `closed_at`, `closed_by`, `closed_late`; chain facts from `AmendableMixin`: nullable self-FK `amends_id`, `superseded_at`, and `chain_closed_late`; `tax_calendar_entry_id` FK to `tax_calendar_entries.id` non-null with `RESTRICT`; nullable due-date snapshot fields `due_date_original`, `due_date_effective`, `due_date_override_reason`; timestamps; soft-delete fields `deleted_at`, `deleted_by`. There is no stored `is_amendment` flag: `is_amendment` is derived as `amends_id IS NOT NULL`. `closed_late` answers for this row; `chain_closed_late` preserves the period's original lateness across corrections. Cite: `backend/app/vat/models/vat_work_item.py:45-120`, `backend/app/common/obligation_chain.py:49-92`.

The slot index is unique on `(client_record_id, period)` only for rows that are not deleted, not amendments, and not canceled: `deleted_at IS NULL AND amends_id IS NULL AND status <> 'canceled'`. A second partial unique index on `amends_id` prevents a chain from forking. Chain-tip read indexes include `superseded_at IS NULL`. Cite: `backend/app/vat/models/vat_work_item.py:135-177`.

**`VatInvoice`** (`vat_invoices`) stores source document totals. Columns: `id`; `work_item_id` FK to `vat_work_items.id` with cascade delete, non-null; `created_by` FK to `users.id` non-null; optional `business_activity_id` FK to `businesses.id`; `invoice_type`; optional `document_type`; `invoice_number`; `invoice_date` as a business `date`; `counterparty_name`; optional external `counterparty_id` and `counterparty_id_type` (business ID / personal ID / passport / foreign ID, not an internal FK); positive `net_amount` and `vat_amount`; optional `expense_category`; `rate_type`; `deduction_rate`; `is_exceptional`; `created_at` as a UTC timestamp. API responses expose `gross_amount` as a computed `net_amount + vat_amount` field so clients can verify the gross-to-net split. Cite: `backend/app/vat/models/vat_invoice.py:40-99` and `backend/app/vat/schemas/vat_invoice_schema.py:88`.

`VatInvoice` has a unique constraint on `(work_item_id, invoice_type, invoice_number)` and indexes on `(work_item_id, invoice_type)` and `invoice_date`. Cite: `backend/app/vat/models/vat_invoice.py:101-110`.

**Audit storage:** VAT no longer writes or reads `VatAuditLog` in the active product flow. VAT mutations write `EntityAuditLog` through `EntityAuditWriter` and the generic audit policy. The legacy `vat_audit_logs` table/model was dropped by the Phase 9 cleanup migration; no current VAT route/service/seed reads or writes it.

## Enums / statuses

**VAT has no status enum of its own.** It runs the shared six-stage `ObligationStatus`
(`backend/app/common/enums.py`), and the transition rules live once in
`backend/app/common/obligation_lifecycle.py` — forward one stage at a time, backward one stage
with a reason, `submitted` has no outgoing transition. See `docs/tax-lifecycle-refactor-plan.md`
§4.1.

| Stage | Value | Label | What it means for VAT |
|---|---|---|---|
| 1 | `awaiting_input` | ממתין לחומר | waiting on the client's documents |
| 2 | `input_received` | החומר התקבל | documents in hand, no invoice entered yet |
| 3 | `in_progress` | בעבודה | data entry under way |
| 4 | `awaiting_verification` | ממתין לאימות | entered and ready, waiting on an advisor |
| 5 | `submitted` | הוגש | filed; locked, correctable only by an amendment |
| — | `canceled` | בוטל | off-ladder, terminal |

### Action keys are not status names

`available_actions` publishes **action keys**, built by
`backend/app/actions/services/vat_report_actions.py`. A key names the act, not the stage it lands
on, and the two deliberately differ — `ready_for_review` moves an item *to*
`awaiting_verification`. The frontend must match on the key. Matching on the target stage name is
exactly the defect that hid the "שלח לבדיקה" button for every `in_progress` period until
2026-07-30.

| Key | Offered when | Role |
|---|---|---|
| `materials_complete` | stage 1 | any |
| `add_invoice` | stages 2, 3, 4 | any |
| `ready_for_review` | stage 3 | any |
| `file_vat_return` | stage 4 | advisor only |
| `send_back` | stage 4 | advisor only |
| `create_amendment` | stage 5, and only on the tip of the chain (`superseded_at IS NULL`) | advisor only |

Role filtering happens **in the builder**, so a secretary genuinely receives a shorter list and an
empty action row is a legitimate state. The frontend must not re-check the role — that would be a
second source, free to drift.

**Resolved statuses.** `RESOLVED_OBLIGATION_STATUSES = {submitted, canceled}`
(`backend/app/common/enums.py`) is the single answer to "does this obligation need further work?"
for all three domains — the question the tax calendar asks. `is_obligation_resolved` reads it, and
so does every SQL query asking the same thing (`vat_compliance_repository`,
`vat_work_item_grouped_repository`, `vat_work_item_query_repository`). One published set is the
Python/SQL twin, so the two forms cannot drift.

`canceled` belongs in the set — a cancelled period is not outstanding work. It was previously
omitted from the Python set while the SQL side excluded it, so a cancelled period read **open** on
the grouped tax calendar and **closed** on the compliance list. Do not confuse this with the
closed-only checks (`OBLIGATION_LOCKED`), which answer the different question "was it actually
filed?".

Other VAT enums:

| Enum | Values | Source |
|------|--------|--------|
| `CounterpartyIdType` | `il_business`, `il_personal`, `foreign`, `anonymous` | `backend/app/vat/models/vat_enums.py:15-19` |
| `InvoiceType` | `income`, `expense` | `backend/app/vat/models/vat_enums.py:22-24` |
| `ExpenseCategory` | `inventory`, `office`, `travel`, `professional_services`, `equipment`, `rent`, `salary`, `marketing`, `vehicle`, `fuel`, `vehicle_maintenance`, `vehicle_insurance`, `vehicle_leasing`, `tolls_and_parking`, `entertainment`, `gifts`, `communication`, `insurance`, `maintenance`, `municipal_tax`, `utilities`, `postage_and_shipping`, `bank_fees`, `mixed_expense` | `backend/app/vat/models/vat_enums.py:27-51` |

**Domain split:** `ExpenseCategory` is the granular VAT/data-entry classification (24 values). It is separate from annual reports' `ExpenseCategoryType` (12 values), which is a higher-level recognition category for income-tax reporting. The two are related by the `_VAT_TO_ANNUAL` mapping in `backend/app/annual_reports/services/annual_report_vat_import_service.py`. Every new `ExpenseCategory` value must be added to that mapping. See `docs/domains/annual-reports.md` for the annual-report side.
| `VatRateType` | `standard`, `exempt`, `zero_rate` | `backend/app/vat/models/vat_enums.py:54-57` |
| `DocumentType` | `tax_invoice`, `transaction_invoice`, `receipt`, `consolidated`, `self_invoice`, `credit_note` | `backend/app/vat/models/vat_enums.py:60-66` |
| `VatType` | `monthly`, `bimonthly`, `exempt` | `backend/app/common/enums.py:12-17` |
| `SubmissionMethod` | `online`, `manual`, `representative` | `backend/app/common/enums.py:6-9` |


## Lifecycle

**This domain runs the shared obligation lifecycle.** VAT, advance payments and
annual reports are three views of one thing — a client owes an obligation for a
period, works it through, and settles it by a deadline — and they now share one
status enum and one transition graph:

| # | `ObligationStatus` | Label |
|---|---|---|
| 1 | `awaiting_input` | ממתין לחומר |
| 2 | `input_received` | החומר התקבל |
| 3 | `in_progress` | בעבודה |
| 4 | `awaiting_verification` | ממתין לאימות |
| 5 | `submitted` | הוגש |
| — | `canceled` | בוטל (off-ladder, reachable from any unlocked stage) |

Rules, enforced once in `app/common/obligation_lifecycle.py`:

- **Forward one stage at a time.** An event may perform consecutive transitions and
  records each; it never skips a stage's meaning.
- **Backward one stage at a time, always with a reason** (`OBLIGATION.TRANSITION_REASON_REQUIRED`).
- **`submitted` has no outgoing transition** (`OBLIGATION.LOCKED`). Correcting a
  submitted obligation creates a separate amendment record through `POST /amend`.
- **Cancel from any unlocked stage**, and `canceled` is terminal.

`RESOLVED_OBLIGATION_STATUSES` = `{submitted, canceled}` is the single answer to
"does this need further work?", read by both the Python predicate and every SQL
query. `OBLIGATION_STATUS_LABELS` is the single Hebrew vocabulary.

## Domain rules & invariants

**Which periods are owed.** `app/common/obligation_plan.py` is the single answer. It is narrowed by the client's configured frequency **and** by that obligation type's liability range on `LegalEntity` — per type, because the types move independently. A period is owed when it *intersects* the range, so a period the client was liable for on any of its days is created in full. NULL on either side is unbounded. See `docs/domains/clients.md` § Liability ranges.

- Work items are anchored to `client_record_id`, not directly to `legal_entity_id`. The service resolves the active client and legal entity through `VatClientContextService`. Cite: `backend/app/vat/services/vat_intake_service.py`.
- Page-level selected-client filters use `client_record_id` for exact `ClientRecord` matching on list, grouped, group-items, and status-summary endpoints. `client_name` is retained only as a free-text/fuzzy API filter.
- Closed or frozen clients cannot create new VAT work items. Cite: `backend/app/vat/services/vat_intake_service.py`.
- Effective VAT frequency is derived from legal entity type and `vat_reporting_frequency`: `OSEK_PATUR` and `EMPLOYEE` resolve to `exempt`; otherwise the configured VAT frequency is used, falling back to `monthly`. Cite: `backend/app/vat/vat_type_resolver.py`.
- Exempt clients cannot create periodic VAT work items. Cite:
  `backend/app/vat/services/vat_intake_service.py` (`_assert_client_reports_vat`).
- Bi-monthly clients cannot create work items for even start months. This rule is **not** VAT's:
  it is enforced once by `TaxCalendarMaterializationService._validate_period_alignment` and raises
  `TAX_CALENDAR.INVALID_PERIOD_ALIGNMENT`. VAT's own copy and its
  `VAT.INVALID_PERIOD_FOR_FREQUENCY` code were retired — see `docs/domains/tax-calendar.md`.
- Creation asks the slot-occupancy query, whose predicate matches the partial unique index: a canceled original frees the period, an amendment never occupies it, and a superseded original continues to occupy it. A duplicate slot raises `VAT.CONFLICT`. Cite: `backend/app/vat/services/vat_intake_service.py`, `backend/app/common/obligation_chain.py:168-199`.
- Creating a work item materializes or reuses a `TaxCalendarEntry`, stores its FK, and snapshots `due_date_original` and `due_date_effective` from the calendar due date. Cite: `backend/app/vat/services/vat_intake_service.py`.
- Creating with `mark_pending=True` requires `pending_materials_note` and starts at `awaiting_input`; otherwise it starts at `input_received`. Cite: `backend/app/vat/services/vat_intake_service.py`.
- `materials-complete` moves `awaiting_input` to `input_received`. Cite: `backend/app/vat/services/vat_intake_service.py`.
- Filed work items are immutable for invoice create/update/delete. Cite: `backend/app/vat/vat_data_entry_common.py`, `backend/app/vat/services/vat_data_entry_invoices_service.py`, `backend/app/vat/services/vat_data_entry_invoice_update_service.py`, `backend/app/vat/services/vat_data_entry_invoice_delete_service.py`.
- Adding the first invoice from `input_received` auto-transitions the item to `in_progress`. Adding invoices is otherwise allowed only in `in_progress` or `awaiting_verification`. Cite: `backend/app/vat/services/vat_data_entry_invoices_service.py`.
- Invoice gross amount must be positive; VAT is split from gross using the annual configured VAT rate, with `exempt` and `zero_rate` producing zero VAT. Cite: `backend/app/vat/services/vat_data_entry_invoices_service.py`, `backend/app/vat/vat_amounts.py`.
- Expense invoices require `expense_category`; expense tax invoices require `counterparty_id`; negative VAT and non-positive net amounts are rejected. Cite: `backend/app/vat/vat_data_entry_common.py`.
- `business_activity_id` on invoice create must belong to the same legal entity as the work item's client record. Cite: `backend/app/vat/services/vat_data_entry_invoices_service.py`.
- OSEK PATUR income invoices are checked against the annual ceiling; crossing the warning threshold returns a non-blocking warning, crossing the ceiling raises an error. Cite: `backend/app/vat/vat_data_entry_common.py`.
- Credit notes are stored with positive amounts but contribute as negative values during aggregation. Cite: `backend/app/vat/repositories/vat_invoice_aggregation_repository.py:18-27`.
- Work-item VAT totals are recalculated after invoice create/update/delete. Output VAT includes standard-rate income VAT; input VAT is expense VAT multiplied by `deduction_rate`; `net_vat = output - input`. Cite: `backend/app/vat/vat_data_entry_common.py`, `backend/app/vat/repositories/vat_invoice_aggregation_repository.py`, `backend/app/vat/repositories/vat_work_item_write_repository.py`.
- `ready-for-review` accepts `in_progress` and moves to `awaiting_verification`; `send-back` requires a non-empty correction note and moves back to `in_progress`. Both use the shared transition graph. Cite: `backend/app/vat/services/vat_data_entry_status_service.py`.
- Filing moves `awaiting_verification` to `submitted`, permits an optional override only with justification, and writes final filing fields plus generic `EntityAuditLog` entries. If an override amount is supplied, `vat_work_item.amount_overridden` is written before `vat_work_item.filed`; otherwise only `vat_work_item.filed` is written. Cite: `backend/app/vat/services/vat_filing_service.py`.
- Filing requires `assigned_to` to be non-null; filing an unassigned item raises `VAT.ASSIGNEE_REQUIRED`. Cite: `backend/app/vat/services/vat_filing_service.py`.
- **`GET /work-items/{item_id}/readiness` has exactly one reachable gate: the assignee** (verified 2026-07-30). It checks `assigned_to is None` and `net_vat is None`, but `net_vat` is `Numeric(12,2) NOT NULL DEFAULT 0.00` (`backend/app/vat/models/vat_work_item.py:75`), so for any persisted row the second condition cannot hold. The final-amount branch is therefore unreachable in both paths — as a readiness issue and as the `VAT.MISSING_FINAL_AMOUNT` error at filing. Cite: `backend/app/vat/services/vat_filing_service.py:43-46,80-83`.
- **A work item with no invoices can be filed** (verified 2026-07-30). Zero is a legitimate VAT figure, but there is no gate on having any data at all, and the state is reachable: adding the first invoice auto-advances to `in_progress`, invoice delete is permitted in every status except `submitted` (`assert_editable` tests only `SUBMITTED`), deleting recalculates the totals back to `0.00` without moving the status back, and neither `ready-for-review` nor `/file` counts invoices. The system cannot distinguish a deliberate no-activity period from an item emptied by mistake, from invoices that offset to zero, from a period nobody has entered yet — and `net_vat == 0` cannot tell them apart either, so any future gate must count invoices. Product decision open as **O-10** in `docs/tax-lifecycle-refactor-plan.md`. Cite: `backend/app/vat/vat_data_entry_common.py:32-35`, `backend/app/vat/services/vat_data_entry_invoices_service.py:153`, `backend/app/vat/services/vat_data_entry_invoice_delete_service.py:58`, `backend/app/vat/services/vat_data_entry_status_service.py:37`.
- **After creation, `PATCH /work-items/{item_id}` (advisor or secretary) is the only route that sets `assigned_to` — and no frontend caller exists** (verified 2026-07-30). The create request accepts the field too, so a work item *can* be created assigned; the create form does not send it, and the detail screen hides the assignee row when it is null instead of showing "unassigned". The gate above is therefore server-enforced and unsatisfiable from the UI; the planned fix is recorded as the W3 follow-up in `docs/tax-lifecycle-refactor-progress.md`.
- Generic work-item PATCH is limited to operational metadata: `assigned_to` and `pending_materials_note`. It uses partial-update semantics, so omitted fields are not changed and explicit `null` clears nullable metadata. It does not update status, period, client identity, VAT totals, filing fields, amendment fields, or calendar snapshot fields.
- Submitted work items reject generic metadata PATCH and DELETE; submitted VAT periods are records of filing and must not be hidden through delete.
- Work-item DELETE is a soft delete for an unlocked standalone mistaken obligation. It sets `deleted_at`, `deleted_by`, and `updated_at`, preserves invoices and generic audit history, and writes `vat_work_item.deleted`. An amendment cannot use this path: `assert_deletable` returns 409 `OBLIGATION.AMENDMENT_NOT_DELETABLE`.
- `POST /work-items/{id}/amend` is advisor-only and accepts only a submitted chain tip. It locks the original, copies the row and every invoice, clears row identity, soft-delete state, closing/submission facts and due dates, links the new row with `amends_id`, stamps the original's `superseded_at`, carries `chain_closed_late`, and opens the correction at `in_progress`. A record may have at most one direct amendment, so the chain is a line and cannot cycle. Cite: `backend/app/vat/services/vat_amendment_service.py`, `backend/app/vat/repositories/vat_work_item_write_repository.py:212-235`.
- Lists, counts, turnover reads, invoice aggregates, and client summaries use the chain-tip scope (`deleted_at IS NULL AND superseded_at IS NULL`), so a correction chain normally contributes one row. `GET /chain` is the explicit historical read and includes superseded links and withdrawn corrections.
- `POST /work-items/{id}/withdraw` is advisor-only. It accepts an amendment that is not submitted and is still the chain tip, soft-deletes it, clears the original's `superseded_at`, returns the restored original, and keeps the withdrawn correction visible only in `GET /chain` with `is_withdrawn=true`. A submitted correction is corrected by another amendment, not withdrawn. Cite: `backend/app/vat/services/vat_amendment_service.py:94-164`, `backend/app/common/obligation_chain.py:286-347`.
- `business_activity_id` on invoice update must belong to the same legal entity as the work item's client record; mismatched or non-existent IDs return `BUSINESS_ACTIVITY.NOT_FOUND` (404, no existence leak). Cite: `backend/app/vat/services/vat_data_entry_invoice_update_service.py`.
- `due_date_original` is immutable after it is first set. If `due_date_effective` differs from `due_date_original`, a non-empty `due_date_override_reason` is required. Cite: `backend/app/vat/models/vat_due_date_snapshot_events.py`.
- VAT audit actions are namespaced strings from `backend/app/audit/audit_constants.py`, not enums. Work-item actions include `vat_work_item.created`, `vat_work_item.status_changed`, `vat_work_item.filed`, `vat_work_item.amount_overridden`, `vat_work_item.updated`, `vat_work_item.deleted`, `vat_work_item.amended`, and `vat_work_item.amendment_withdrawn`. Invoice actions are `vat_invoice.created`, `vat_invoice.updated`, `vat_invoice.amount_changed`, and `vat_invoice.deleted`. Every VAT audit row must carry `metadata_json.client_record_id`; invoice rows also carry `metadata_json.vat_work_item_id` and invoice context. Cite: `backend/app/vat/vat_audit.py`, `backend/app/audit/audit_constants.py`, `backend/app/audit/audit_write_policy.py`.

## Error codes

The `VAT.REASON` codes this domain raises. Registry: `docs/backend/error-codes.md`.

| Code | Status mapping | Raised when |
|------|----------------|-------------|
| `VAT.NOT_FOUND` | 404 via `NotFoundError` | Work item, client, period option, or export target is not found |
| `VAT.CLIENT_RECORD_NOT_FOUND` | 404 via `NotFoundError` | Work item's client record is missing during invoice create |
| `VAT.CLIENT_CLOSED` | 400 via `AppError` | Client is closed for **invoice** create (`vat_data_entry_invoices_service.py`). Work-item create answers from the shared client guard with 409 `CLIENT_RECORD.CLOSED` |
| `VAT.CLIENT_EXEMPT` | 400 via `AppError` | Exempt client is used for periodic VAT work item or period options |
| `VAT.CONFLICT` | 409 via `ConflictError` | Duplicate active work item or duplicate invoice number |
| `VAT.PENDING_NOTE_REQUIRED` | 400 via `AppError` | `mark_pending=True` without `pending_materials_note` |
| `VAT.INVALID_TRANSITION` | 400 via `AppError` | Illegal status transition |
| `VAT.INVALID_STATUS` | 400 via `AppError` | Invoice add attempted from a status not allowed for add |
| `VAT.FILED_IMMUTABLE` | 400 via `AppError` | Invoice mutation, generic metadata update, or delete attempted after filing |
| `VAT.NET_NOT_POSITIVE` | 400 via `AppError` | Non-positive net/gross amount |
| `VAT.NEGATIVE_VAT` | 400 via `AppError` | Negative VAT amount |
| `VAT.EXPENSE_CATEGORY_REQUIRED` | 400 via `AppError` | Expense invoice missing category |
| `VAT.COUNTERPARTY_ID_REQUIRED` | 400 via `AppError` | Expense tax invoice missing supplier ID |
| `VAT.OSEK_PATUR_CEILING_EXCEEDED` | 400 via `AppError` | OSEK PATUR annual turnover would exceed configured ceiling |
| `VAT.JUSTIFICATION_REQUIRED` | 400 via `AppError` | Missing correction note or override justification |
| `VAT.ASSIGNEE_REQUIRED` | 400 via `AppError` | Filing attempted when `assigned_to` is null |
| `VAT.MISSING_FINAL_AMOUNT` | 400 via `AppError` | `net_vat` is null and no override amount supplied at filing. **Unreachable as of 2026-07-30** — the column is `NOT NULL DEFAULT 0.00`, so no persisted row can trigger it. Whether it is replaced by an invoice-count gate or retired is part of O-10 |

Related codes raised from this module but owned by another namespace:
- `OBLIGATION.NOT_CLOSED` — amendment creation requested from a record that is not submitted.
- `OBLIGATION.ALREADY_AMENDED` — the requested record already has a correction, or withdrawal is requested from a non-tip link.
- `OBLIGATION.AMENDMENT_NOT_DELETABLE` — plain DELETE requested for an amendment.
- `OBLIGATION.NOT_AN_AMENDMENT` — withdrawal requested for a standalone record.
- `OBLIGATION.LOCKED` — withdrawal requested for a submitted amendment.
- `BUSINESS_ACTIVITY.WRONG_CLIENT` — invoice create: activity does not belong to the work item's legal entity. Cite: `backend/app/vat/services/vat_data_entry_invoices_service.py`.
- `BUSINESS_ACTIVITY.NOT_FOUND` — invoice update: `business_activity_id` not found or belongs to a different legal entity (404, no existence leak). Cite: `backend/app/vat/services/vat_data_entry_invoice_update_service.py`.

## Known issues

Found 2026-07-30, all open and unstarted. Described as current behavior above; the planned work
is tracked in `docs/tax-lifecycle-refactor-progress.md` (W3 follow-up) and `O-10` in
`docs/tax-lifecycle-refactor-plan.md`.

- **The assignee filing gate cannot be satisfied from the UI.** `available_actions` publishes
  `file_vat_return` on role and status alone, so the button is offered, the filing dialog is
  filled, and the 400 arrives last. There is no assignee control on any VAT screen.
- **The final-amount gate is unreachable**, in readiness and at filing alike.
- **A period with no invoices can be filed**, and the four ways a period can read as zero are
  indistinguishable.

## Resolved issues

- **F-007, F-008, F-009, F-010** (2026-06-04): Resolved.

## Decisions (preserved)

1. **VAT workflow objects are client-record scoped.** `VatWorkItem` links to `client_record_id`, not directly to `legal_entity_id`; any legal-entity join goes through `ClientRecord`. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat/models/vat_work_item.py:46-48`.
2. **VAT reporting frequency is independent from advance-payment frequency.** VAT work-item creation uses `vat_reporting_frequency` / `VatType` only; it does not derive from advance-payment settings. Confirmed in `backend/app/vat/vat_type_resolver.py`.
3. **Period is the business identity; corrections form a chain.** The original slot is unique by `(client_record_id, period)` among non-deleted, non-amendment, non-canceled rows. Corrections share the period and link through `amends_id`; normal reads expose only chain tips. Confirmed in `backend/app/vat/models/vat_work_item.py:55-59,135-177` and `backend/app/common/obligation_chain.py`.
4. **TaxCalendarEntry is the regulatory deadline source, but work items store snapshots.** Work-item creation materializes a VAT calendar entry and snapshots original/effective due dates. Confirmed in `backend/app/vat/services/vat_intake_service.py`.
5. **`due_date_effective` is the intended date for overdue/reminder logic.** The code stores and enforces effective-date snapshot integrity; no current endpoint updates it. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat/models/vat_due_date_snapshot_events.py`.
6. **Filed VAT periods are immutable for invoice mutation.** Confirmed in `backend/app/vat/vat_data_entry_common.py`.
7. **Credit notes reverse totals without storing negative document amounts.** Confirmed in `backend/app/vat/models/vat_invoice.py:79-81` and `backend/app/vat/repositories/vat_invoice_aggregation_repository.py:18-27`.
8. **Business activity is optional tagging, not the VAT owner.** `business_activity_id` may be null on invoices, and VAT work items remain owned by the client record. Confirmed in `backend/app/vat/models/vat_invoice.py:52-56`.
9. **VAT audit is entity-scoped.** Work-item lifecycle history belongs to `vat_work_item`; invoice add/update/delete/amount-change history belongs to `vat_invoice` with the owning work item in `metadata_json.vat_work_item_id`. A work-item History tab that reads only `/audit/vat_work_item/{id}` will not show invoice events. Product follow-up for a possible aggregated VAT-period history view is tracked in `docs/vat-history-product-followup.md`.

## Future / planned

Explicitly not-yet-implemented behavior. Never describe as current.

- Dedicated due-date override endpoint with reason, permission checks, and terminal-state guards. Current code has model/event validation but no VAT API route for changing `due_date_effective`.
- Full use of `period_months_count` on `VatWorkItem` instead of `period_type`, if the project later aligns VAT work items fully with `TaxCalendarEntry`.
- Zero-report modeling beyond a filed work item with zero totals. Legacy docs require explicit zero-report support, but current code has no separate zero-report status or flag.
- Allocation-number validation for invoices above the legal threshold. Legacy docs describe it; current `VatInvoice` has no allocation-number field.
- Separation of capital input VAT from other input VAT. Current code has `expense_category` and aggregated input totals, but no distinct capital-input VAT return field.
- Cash-basis/accrual-basis period assignment rules. Current invoice period is inherited from the work item; there is no implemented timing-basis engine in this module.
- OSEK PATUR annual VAT declaration workflow. Periodic work-item creation rejects exempt clients; the annual declaration is not implemented here.

## Historical notes

Legacy VAT domain material archived at `docs/archive/vat-reports-legacy.md`.

Original legacy VAT material is archived at `docs/archive/vat-reports-legacy.md`.
