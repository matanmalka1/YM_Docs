## Scope
This file owns only:
- Canonical current-state documentation for the vat_reports domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# VAT Reports

The VAT reports domain manages period-based VAT work items for a `ClientRecord`: material intake, invoice data entry, review, filing, audit history, client summaries, and VAT exports. The implemented aggregate is `VatWorkItem`, with `VatInvoice` rows as source documents and `VatAuditLog` rows as append-only workflow history.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All paths exist in `backend/openapi.json` (`backend/openapi.json:9757`, `backend/openapi.json:9927`, `backend/openapi.json:9975`, `backend/openapi.json:10095`, `backend/openapi.json:10210`, `backend/openapi.json:10258`, `backend/openapi.json:10316`, `backend/openapi.json:10374`, `backend/openapi.json:10476`, `backend/openapi.json:10578`, `backend/openapi.json:10642`, `backend/openapi.json:10707`, `backend/openapi.json:10795`, `backend/openapi.json:10842`, `backend/openapi.json:10912`, `backend/openapi.json:10982`, `backend/openapi.json:11029`).

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/vat/work-items` | Create a VAT work item for a client period |
| `POST` | `/api/v1/vat/work-items/{item_id}/materials-complete` | Move pending material intake to material received |
| `POST` | `/api/v1/vat/work-items/{item_id}/invoices` | Add an income or expense invoice to a work item |
| `GET` | `/api/v1/vat/work-items/{item_id}/invoices` | List invoices for a work item, optionally by invoice type |
| `PATCH` | `/api/v1/vat/work-items/{item_id}/invoices/{invoice_id}` | Update an invoice on an editable work item |
| `DELETE` | `/api/v1/vat/work-items/{item_id}/invoices/{invoice_id}` | Delete an invoice from an editable work item |
| `POST` | `/api/v1/vat/work-items/{item_id}/ready-for-review` | Move data entry to review |
| `POST` | `/api/v1/vat/work-items/{item_id}/send-back` | Advisor sends a reviewed item back for correction |
| `POST` | `/api/v1/vat/work-items/{item_id}/file` | Advisor files the VAT return |
| `GET` | `/api/v1/vat/work-items/groups` | List grouped work-item summaries |
| `GET` | `/api/v1/vat/work-items/groups/{group_key}/items` | List items in a grouped due-date bucket |
| `GET` | `/api/v1/vat/work-items/lookup` | Look up one item by `client_record_id` and `period` |
| `GET` | `/api/v1/vat/clients/{client_record_id}/period-options` | Return valid period options for a client |
| `GET` | `/api/v1/vat/work-items/status-summary` | Count work items by status |
| `GET` | `/api/v1/vat/work-items/{item_id}` | Get one enriched work item |
| `GET` | `/api/v1/vat/clients/{client_record_id}/work-items` | List one client's work items |
| `GET` | `/api/v1/vat/work-items` | List work items across clients |
| `GET` | `/api/v1/vat/work-items/{item_id}/audit` | Get audit trail for a work item |
| `GET` | `/api/v1/vat/clients/{client_record_id}/summary` | Get client-level VAT period and annual aggregates |
| `GET` | `/api/v1/vat/clients/{client_record_id}/export` | Export a client's VAT data as Excel or PDF |

Router sources: `backend/app/vat_reports/api/routes_intake.py:14-22`, `backend/app/vat_reports/api/routes_data_entry.py:16-104`, `backend/app/vat_reports/api/routes_status.py:16-50`, `backend/app/vat_reports/api/routes_filing.py:13-24`, `backend/app/vat_reports/api/routes_grouped.py:17-46`, `backend/app/vat_reports/api/routes_queries.py:25-158`, `backend/app/vat_reports/api/routes_client_summary.py:12-38`.

## Model & fields

**`VatWorkItem`** (`vat_work_items`) is the root VAT workflow record. Columns: `id`; `client_record_id` FK to `client_records.id` non-null; `created_by` FK to `users.id` non-null; `assigned_to` FK to `users.id` nullable; `period` `String(7)` non-null; `period_type` `VatType` non-null; `status` `VatWorkItemStatus` non-null; `pending_materials_note` nullable; `total_output_vat`, `total_input_vat`, `net_vat`, `total_output_net`, `total_input_net` non-null `Numeric(12,2)` totals; filing fields `final_vat_amount`, `is_overridden`, `override_justification`, `submission_method`, `filed_at`, `filed_by`, `submission_reference`; amendment fields `is_amendment`, nullable `amends_item_id` FK to `vat_work_items.id`; `tax_calendar_entry_id` FK to `tax_calendar_entries.id` non-null with `RESTRICT`; due-date snapshot fields `due_date_original`, `due_date_effective`, `due_date_override_reason`; timestamps; soft-delete fields `deleted_at`, `deleted_by`. Cite: `backend/app/vat_reports/models/vat_work_item.py:42-119`.

Indexes include active unique `(client_record_id, period) WHERE deleted_at IS NULL`, status, period, calendar-entry, turnover lookup, and active due/client indexes. Cite: `backend/app/vat_reports/models/vat_work_item.py:134-165`.

**`VatInvoice`** (`vat_invoices`) stores source document totals. Columns: `id`; `work_item_id` FK to `vat_work_items.id` with cascade delete, non-null; `created_by` FK to `users.id` non-null; optional `business_activity_id` FK to `businesses.id`; `invoice_type`; optional `document_type`; `invoice_number`; `invoice_date`; `counterparty_name`; optional `counterparty_id` and `counterparty_id_type`; positive `net_amount` and `vat_amount`; optional `expense_category`; `rate_type`; `deduction_rate`; `is_exceptional`; `created_at`. API responses expose `gross_amount` as a computed `net_amount + vat_amount` field so clients can verify the gross-to-net split. Cite: `backend/app/vat_reports/models/vat_invoice.py:40-99` and `backend/app/vat_reports/schemas/vat_invoice_schema.py:88`.

`VatInvoice` has a unique constraint on `(work_item_id, invoice_type, invoice_number)` and indexes on `(work_item_id, invoice_type)` and `invoice_date`. Cite: `backend/app/vat_reports/models/vat_invoice.py:101-110`.

**`VatAuditLog`** (`vat_audit_logs`) is append-only workflow history. Columns: `id`; `work_item_id` FK to `vat_work_items.id` non-null; `performed_by` FK to `users.id` non-null; free-string `action`; optional `old_value`, `new_value`, `note`; nullable `invoice_id` FK to `vat_invoices.id` with `SET NULL`; `performed_at`. Cite: `backend/app/vat_reports/models/vat_audit_log.py:24-46`.

## Enums / statuses

`VatWorkItemStatus` values (`backend/app/vat_reports/models/vat_enums.py:6-12`):

| Value |
|-------|
| `pending_materials` |
| `material_received` |
| `data_entry_in_progress` |
| `ready_for_review` |
| `filed` |
| `canceled` |

Other VAT enums:

| Enum | Values | Source |
|------|--------|--------|
| `CounterpartyIdType` | `il_business`, `il_personal`, `foreign`, `anonymous` | `backend/app/vat_reports/models/vat_enums.py:15-19` |
| `InvoiceType` | `income`, `expense` | `backend/app/vat_reports/models/vat_enums.py:22-24` |
| `ExpenseCategory` | `inventory`, `office`, `travel`, `professional_services`, `equipment`, `rent`, `salary`, `marketing`, `vehicle`, `fuel`, `vehicle_maintenance`, `vehicle_insurance`, `vehicle_leasing`, `tolls_and_parking`, `entertainment`, `gifts`, `communication`, `insurance`, `maintenance`, `municipal_tax`, `utilities`, `postage_and_shipping`, `bank_fees`, `mixed_expense` | `backend/app/vat_reports/models/vat_enums.py:27-51` |
| `VatRateType` | `standard`, `exempt`, `zero_rate` | `backend/app/vat_reports/models/vat_enums.py:54-57` |
| `DocumentType` | `tax_invoice`, `transaction_invoice`, `receipt`, `consolidated`, `self_invoice`, `credit_note` | `backend/app/vat_reports/models/vat_enums.py:60-66` |
| `VatType` | `monthly`, `bimonthly`, `exempt` | `backend/app/common/enums.py:12-17` |
| `SubmissionMethod` | `online`, `manual`, `representative` | `backend/app/common/enums.py:6-9` |

## Domain rules & invariants

- Work items are anchored to `client_record_id`, not directly to `legal_entity_id`. The service resolves the active client and legal entity through `VatClientContextService`. Cite: `backend/app/vat_reports/services/intake.py:62-70`.
- Closed or frozen clients cannot create new VAT work items. Cite: `backend/app/vat_reports/services/intake.py:65-68`.
- Effective VAT frequency is derived from legal entity type and `vat_reporting_frequency`: `OSEK_PATUR` and `EMPLOYEE` resolve to `exempt`; otherwise the configured VAT frequency is used, falling back to `monthly`. Cite: `backend/app/vat_reports/services/vat_type_resolver.py:6-18`.
- Exempt clients cannot create periodic VAT work items. Bi-monthly clients cannot create work items for even start months. Cite: `backend/app/vat_reports/services/intake.py:36-48`.
- Active duplicates are blocked by service check and partial unique index on `(client_record_id, period) WHERE deleted_at IS NULL`. Cite: `backend/app/vat_reports/services/intake.py:72-80`, `backend/app/vat_reports/models/vat_work_item.py:134-142`.
- Creating a work item materializes or reuses a `TaxCalendarEntry`, stores its FK, and snapshots `due_date_original` and `due_date_effective` from the calendar due date. Cite: `backend/app/vat_reports/services/intake.py:92-111`.
- Creating with `mark_pending=True` requires `pending_materials_note`; otherwise initial status is `material_received`. Cite: `backend/app/vat_reports/services/intake.py:82-90`.
- `materials-complete` only moves `pending_materials` to `material_received`. Cite: `backend/app/vat_reports/services/intake.py:125-155`.
- Filed work items are immutable for invoice create/update/delete. Cite: `backend/app/vat_reports/services/data_entry_common.py:32-35`, `backend/app/vat_reports/services/data_entry_invoices.py:68-72`, `backend/app/vat_reports/services/data_entry_invoice_update.py:55-60`, `backend/app/vat_reports/services/data_entry_invoice_delete.py:35-40`.
- Adding the first invoice from `material_received` auto-transitions the item to `data_entry_in_progress`. Adding invoices is otherwise allowed only in `data_entry_in_progress` or `ready_for_review`. Cite: `backend/app/vat_reports/services/data_entry_invoices.py:141-160`.
- Invoice gross amount must be positive; VAT is split from gross using the annual configured VAT rate, with `exempt` and `zero_rate` producing zero VAT. Cite: `backend/app/vat_reports/services/data_entry_invoices.py:96-100`, `backend/app/vat_reports/services/vat_amounts.py:24-39`.
- Expense invoices require `expense_category`; expense tax invoices require `counterparty_id`; negative VAT and non-positive net amounts are rejected. Cite: `backend/app/vat_reports/services/data_entry_common.py:74-110`.
- `business_activity_id` on invoice create must belong to the same legal entity as the work item's client record. Cite: `backend/app/vat_reports/services/data_entry_invoices.py:88-94`.
- OSEK PATUR income invoices are checked against the annual ceiling; crossing the warning threshold returns a non-blocking warning, crossing the ceiling raises an error. Cite: `backend/app/vat_reports/services/data_entry_common.py:113-138`.
- Credit notes are stored with positive amounts but contribute as negative values during aggregation. Cite: `backend/app/vat_reports/repositories/vat_invoice_aggregation_repository.py:18-27`.
- Work-item VAT totals are recalculated after invoice create/update/delete. Output VAT includes standard-rate income VAT; input VAT is expense VAT multiplied by `deduction_rate`; `net_vat = output - input`. Cite: `backend/app/vat_reports/services/data_entry_common.py:51-60`, `backend/app/vat_reports/repositories/vat_invoice_aggregation_repository.py:29-87`, `backend/app/vat_reports/repositories/vat_work_item_write_repository.py:151-171`.
- `ready-for-review` only accepts `data_entry_in_progress`; `send-back` requires a non-empty correction note and transitions back to `data_entry_in_progress`. Cite: `backend/app/vat_reports/services/data_entry_status.py:17-49`, `backend/app/vat_reports/services/data_entry_status.py:52-90`.
- Filing requires transition to be allowed by `VALID_TRANSITIONS`, permits optional override only with justification, and writes final filing fields plus audit entries. Cite: `backend/app/vat_reports/services/constants.py:7-25`, `backend/app/vat_reports/services/filing.py:46-114`.
- Amendments can point to an existing filed work item for the same client, and amendment cycles are rejected. Cite: `backend/app/vat_reports/services/filing.py:24-44`.
- `due_date_original` is immutable after it is first set. If `due_date_effective` differs from `due_date_original`, a non-empty `due_date_override_reason` is required. Cite: `backend/app/vat_reports/models/due_date_snapshot_events.py:9-46`.
- Audit actions are stored as strings from service constants, not enum values. Cite: `backend/app/vat_reports/models/vat_audit_log.py:3-12`, `backend/app/vat_reports/services/constants.py:27-35`.

## Error codes

The `VAT.REASON` codes this domain raises. Registry: `docs/architecture/error-codes.md`.

| Code | Status mapping | Raised when |
|------|----------------|-------------|
| `VAT.NOT_FOUND` | 404 via `NotFoundError` | Work item, client, period option, or export target is not found |
| `VAT.CLIENT_RECORD_NOT_FOUND` | 404 via `NotFoundError` | Work item's client record is missing during invoice create |
| `VAT.CLIENT_CLOSED` | 400 via `AppError` | Client is closed for work-item create or invoice create |
| `VAT.CLIENT_FROZEN` | 400 via `AppError` | Client is frozen for work-item create |
| `VAT.CLIENT_EXEMPT` | 400 via `AppError` | Exempt client is used for periodic VAT work item or period options |
| `VAT.INVALID_PERIOD_FOR_FREQUENCY` | 400 via `AppError` | Bi-monthly work item uses an even start month |
| `VAT.CONFLICT` | 409 via `ConflictError` | Duplicate active work item or duplicate invoice number |
| `VAT.PENDING_NOTE_REQUIRED` | 400 via `AppError` | `mark_pending=True` without `pending_materials_note` |
| `VAT.INVALID_TRANSITION` | 400 via `AppError` | Illegal status transition |
| `VAT.INVALID_STATUS` | 400 via `AppError` | Invoice add attempted from a status not allowed for add |
| `VAT.FILED_IMMUTABLE` | 400 via `AppError` | Invoice mutation attempted after filing |
| `VAT.NET_NOT_POSITIVE` | 400 via `AppError` | Non-positive net/gross amount |
| `VAT.NEGATIVE_VAT` | 400 via `AppError` | Negative VAT amount |
| `VAT.EXPENSE_CATEGORY_REQUIRED` | 400 via `AppError` | Expense invoice missing category |
| `VAT.COUNTERPARTY_ID_REQUIRED` | 400 via `AppError` | Expense tax invoice missing supplier ID |
| `VAT.OSEK_PATUR_CEILING_EXCEEDED` | 400 via `AppError` | OSEK PATUR annual turnover would exceed configured ceiling |
| `VAT.JUSTIFICATION_REQUIRED` | 400 via `AppError` | Missing correction note or override justification |

Related code raised from this module but owned by another namespace: `BUSINESS_ACTIVITY.WRONG_CLIENT` when an invoice's business activity does not belong to the work item's client. Cite: `backend/app/vat_reports/services/data_entry_invoices.py:88-94`.

## Known issues

Current code discrepancies found during authoring. These are bugs to fix in code, not intended behavior.

- **Filing does not enforce `assigned_to`.** Preserved invariant INV-09 says `VatWorkItem` should not move to `filed` without `assigned_to`, but `file_vat_return` checks transition, amendment validity, override justification, and final amount only; it never checks `item.assigned_to`. Cite: `backend/docs/domain_decisions_v3.md` INV-09; implementation at `backend/app/vat_reports/services/filing.py:58-99`. Suggested fix: block filing when `assigned_to is None`, or formally retire the invariant.
- **Some VAT service errors do not use the `DOMAIN.REASON` pattern.** Filing raises `AMENDED_ITEM_NOT_FOUND`, `AMENDED_ITEM_WRONG_CLIENT`, `AMENDED_ITEM_NOT_FILED`, `AMENDMENT_CYCLE`, and `MISSING_FINAL_AMOUNT`; invoice update raises `INVALID_NET_AMOUNT`. This conflicts with the error-code format documented in `docs/architecture/error-codes.md`. Cite: `backend/app/vat_reports/services/filing.py:31-40`, `backend/app/vat_reports/services/filing.py:72-73`, `backend/app/vat_reports/services/data_entry_invoice_update.py:76-77`. Suggested fix: rename to `VAT.AMENDED_ITEM_NOT_FOUND`, `VAT.AMENDED_ITEM_WRONG_CLIENT`, `VAT.AMENDED_ITEM_NOT_FILED`, `VAT.AMENDMENT_CYCLE`, `VAT.MISSING_FINAL_AMOUNT`, and `VAT.NET_NOT_POSITIVE`.
- **`is_amendment=True` is accepted without `amends_item_id`.** Amendment validation runs only when `amends_item_id is not None`; `is_amendment` can be stored true with no amended item link. Cite: `backend/app/vat_reports/services/filing.py:55-65`, `backend/app/vat_reports/repositories/vat_work_item_write_repository.py:191-200`. Suggested fix: require `amends_item_id` when `is_amendment` is true.
- **Invoice update does not validate changed `business_activity_id` ownership.** Create validates the activity belongs to the work item's legal entity, but update passes `business_activity_id` through without the same check. Cite: create validation at `backend/app/vat_reports/services/data_entry_invoices.py:88-94`; update fields at `backend/app/vat_reports/services/data_entry_invoice_update.py:81-121`. Suggested fix: apply the create-time ownership check on update when `business_activity_id` is supplied.

## Decisions (preserved)

1. **VAT workflow objects are client-record scoped.** `VatWorkItem` links to `client_record_id`, not directly to `legal_entity_id`; any legal-entity join goes through `ClientRecord`. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat_reports/models/vat_work_item.py:46-48`.
2. **VAT reporting frequency is independent from advance-payment frequency.** VAT work-item creation uses `vat_reporting_frequency` / `VatType` only; it does not derive from advance-payment settings. Confirmed in `backend/app/vat_reports/services/vat_type_resolver.py:6-18`.
3. **Period is the business identity.** Active uniqueness is by `(client_record_id, period)`, and period is stored as `YYYY-MM`. Confirmed in `backend/app/vat_reports/models/vat_work_item.py:52-56` and `backend/app/vat_reports/models/vat_work_item.py:134-142`.
4. **TaxCalendarEntry is the regulatory deadline source, but work items store snapshots.** Work-item creation materializes a VAT calendar entry and snapshots original/effective due dates. Confirmed in `backend/app/vat_reports/services/intake.py:92-111`.
5. **`due_date_effective` is the intended date for overdue/reminder logic.** The code stores and enforces effective-date snapshot integrity; no current endpoint updates it. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat_reports/models/due_date_snapshot_events.py:9-46`.
6. **Filed VAT periods are immutable for invoice mutation.** Confirmed in `backend/app/vat_reports/services/data_entry_common.py:32-35`.
7. **Credit notes reverse totals without storing negative document amounts.** Confirmed in `backend/app/vat_reports/models/vat_invoice.py:79-81` and `backend/app/vat_reports/repositories/vat_invoice_aggregation_repository.py:18-27`.
8. **Business activity is optional tagging, not the VAT owner.** `business_activity_id` may be null on invoices, and VAT work items remain owned by the client record. Confirmed in `backend/app/vat_reports/models/vat_invoice.py:52-56`.
9. **Audit action names are flexible strings.** They are intentionally not enum-backed to avoid migrations for new actions. Confirmed in `backend/app/vat_reports/models/vat_audit_log.py:3-12`.

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

Original legacy files now point here:
- `backend/docs/backend/domains/vat_report/VAT_DEFINITIONS.md`
- `backend/docs/backend/domains/vat_report/source-map.md`
- `backend/docs/backend/domains/vat_report/vat_reports_deep_summary.md`
- `backend/docs/backend/domains/vat_report/vat_reports_domain_summary.md`
