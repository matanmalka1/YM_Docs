## Scope
This file owns only:
- Canonical current-state documentation for the vat domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# VAT

The VAT domain manages period-based VAT work items for a `ClientRecord`: material intake, invoice data entry, review, filing, audit history, client summaries, and VAT exports. The implemented aggregate is `VatWorkItem`, with `VatInvoice` rows as source documents and `VatAuditLog` rows as append-only workflow history.

Last verified against code + backend/openapi.json: 2026-06-14.

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
| `GET` | `/api/v1/vat/work-items/groups` | List grouped work-item summaries; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/groups/{group_key}/items` | List items (thin `VatWorkItemListItem`) in a grouped due-date bucket; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/lookup` | Look up one item by `client_record_id` and `period` |
| `GET` | `/api/v1/vat/clients/{client_record_id}/period-options` | Return valid period options for a client |
| `GET` | `/api/v1/vat/work-items/status-summary` | Count work items by status; selected-client filters use exact `client_record_id`, while `client_name` remains a free-text legacy filter |
| `GET` | `/api/v1/vat/work-items/{item_id}` | Get one enriched work item (full `VatWorkItemResponse`) |
| `GET` | `/api/v1/vat/clients/{client_record_id}/work-items` | List one client's work items (thin `VatWorkItemListItem`); paginated (default `page_size=200`), filterable by `year`, `period`, `status`, `assigned_to`, `due_after`/`due_before` (date, vs `due_date_effective`). Ordered by `period desc, id desc` |
| `GET` | `/api/v1/vat/work-items` | List work items across clients (thin `VatWorkItemListItem`); supports exact `client_record_id` and legacy fuzzy `client_name` filters |
| `GET` | `/api/v1/vat/work-items/{item_id}/audit` | Get audit trail for a work item |
| `GET` | `/api/v1/vat/clients/{client_record_id}/summary` | Get client-level VAT period and annual aggregates |
| `GET` | `/api/v1/vat/clients/{client_record_id}/export` | Export a client's VAT data as Excel or PDF |

Router sources: `backend/app/vat/api/routes_intake.py:14-22`, `backend/app/vat/api/routes_work_items.py`, `backend/app/vat/api/routes_data_entry.py:16-104`, `backend/app/vat/api/routes_status.py:16-50`, `backend/app/vat/api/routes_filing.py:13-24`, `backend/app/vat/api/routes_grouped.py:17-46`, `backend/app/vat/api/routes_queries.py:25-158`, `backend/app/vat/api/routes_client_summary.py:12-38`.

VAT export OpenAPI documents both successful download media types, Excel and PDF, as binary file responses rather than `application/json` with an empty schema (`backend/app/vat/api/routes_client_summary.py:50-57`, `backend/openapi.json`).

List/detail DTO split: the three work-item list endpoints return the thin `VatWorkItemListItem` — only the fields the VAT list/grouped table/cards render (identity, period, status, `net_vat`/`final_vat_amount`/`is_overridden`, the displayed deadline fields, `filed_at`, `updated_at`, `available_actions`). Detail-only fields (raw `total_*` amounts, `override_justification`, `submission_method`/`submission_reference`, `filed_by`/`filed_by_name`, `assigned_to`/`assigned_to_name`, `statutory_deadline`, amendment links, `client_status`, `pending_materials_note`) are served only by `GET /vat/work-items/{item_id}` as the full `VatWorkItemResponse`; the list-row click navigates to the detail page, which refetches by id (`backend/app/vat/schemas/vat_report.py`, `backend/app/vat/api/serializers.py`).

## Model & fields

**`VatWorkItem`** (`vat_work_items`) is the root VAT workflow record. Columns: `id`; `client_record_id` FK to `client_records.id` non-null; `created_by` FK to `users.id` non-null; `assigned_to` FK to `users.id` nullable; `period` `String(7)` non-null; `period_type` `VatType` non-null; `status` `VatWorkItemStatus` non-null; `pending_materials_note` nullable; `total_output_vat`, `total_input_vat`, `net_vat`, `total_output_net`, `total_input_net` non-null `Numeric(12,2)` totals; filing fields `final_vat_amount`, `is_overridden`, `override_justification`, `submission_method`, `filed_at`, `filed_by`, `submission_reference`; amendment fields `is_amendment`, nullable `amends_item_id` FK to `vat_work_items.id`; `tax_calendar_entry_id` FK to `tax_calendar_entries.id` non-null with `RESTRICT`; due-date snapshot fields `due_date_original`, `due_date_effective`, `due_date_override_reason`; timestamps; soft-delete fields `deleted_at`, `deleted_by`. Cite: `backend/app/vat/models/vat_work_item.py:42-119`.

Indexes include active unique `(client_record_id, period) WHERE deleted_at IS NULL`, status, period, calendar-entry, turnover lookup, and active due/client indexes. Cite: `backend/app/vat/models/vat_work_item.py:134-165`.

**`VatInvoice`** (`vat_invoices`) stores source document totals. Columns: `id`; `work_item_id` FK to `vat_work_items.id` with cascade delete, non-null; `created_by` FK to `users.id` non-null; optional `business_activity_id` FK to `businesses.id`; `invoice_type`; optional `document_type`; `invoice_number`; `invoice_date` as a business `date`; `counterparty_name`; optional external `counterparty_id` and `counterparty_id_type` (business ID / personal ID / passport / foreign ID, not an internal FK); positive `net_amount` and `vat_amount`; optional `expense_category`; `rate_type`; `deduction_rate`; `is_exceptional`; `created_at` as a UTC timestamp. API responses expose `gross_amount` as a computed `net_amount + vat_amount` field so clients can verify the gross-to-net split. Cite: `backend/app/vat/models/vat_invoice.py:40-99` and `backend/app/vat/schemas/vat_invoice_schema.py:88`.

`VatInvoice` has a unique constraint on `(work_item_id, invoice_type, invoice_number)` and indexes on `(work_item_id, invoice_type)` and `invoice_date`. Cite: `backend/app/vat/models/vat_invoice.py:101-110`.

**`VatAuditLog`** (`vat_audit_logs`) is append-only workflow history. Columns: `id`; `work_item_id` FK to `vat_work_items.id` non-null; `performed_by` FK to `users.id` non-null; free-string `action`; optional `old_value`, `new_value`, `note`; nullable `invoice_id` FK to `vat_invoices.id` with `SET NULL`; `performed_at`. Cite: `backend/app/vat/models/vat_audit_log.py:24-46`.

## Enums / statuses

`VatWorkItemStatus` values (`backend/app/vat/models/vat_enums.py:6-12`):

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
| `CounterpartyIdType` | `il_business`, `il_personal`, `foreign`, `anonymous` | `backend/app/vat/models/vat_enums.py:15-19` |
| `InvoiceType` | `income`, `expense` | `backend/app/vat/models/vat_enums.py:22-24` |
| `ExpenseCategory` | `inventory`, `office`, `travel`, `professional_services`, `equipment`, `rent`, `salary`, `marketing`, `vehicle`, `fuel`, `vehicle_maintenance`, `vehicle_insurance`, `vehicle_leasing`, `tolls_and_parking`, `entertainment`, `gifts`, `communication`, `insurance`, `maintenance`, `municipal_tax`, `utilities`, `postage_and_shipping`, `bank_fees`, `mixed_expense` | `backend/app/vat/models/vat_enums.py:27-51` |

**Domain split:** `ExpenseCategory` is the granular VAT/data-entry classification (24 values). It is separate from annual reports' `ExpenseCategoryType` (12 values), which is a higher-level recognition category for income-tax reporting. The two are related by the `_VAT_TO_ANNUAL` mapping in `backend/app/annual_reports/services/vat_import_service.py`. Every new `ExpenseCategory` value must be added to that mapping. See `docs/domains/annual-reports.md` for the annual-report side.
| `VatRateType` | `standard`, `exempt`, `zero_rate` | `backend/app/vat/models/vat_enums.py:54-57` |
| `DocumentType` | `tax_invoice`, `transaction_invoice`, `receipt`, `consolidated`, `self_invoice`, `credit_note` | `backend/app/vat/models/vat_enums.py:60-66` |
| `VatType` | `monthly`, `bimonthly`, `exempt` | `backend/app/common/enums.py:12-17` |
| `SubmissionMethod` | `online`, `manual`, `representative` | `backend/app/common/enums.py:6-9` |

## Domain rules & invariants

- Work items are anchored to `client_record_id`, not directly to `legal_entity_id`. The service resolves the active client and legal entity through `VatClientContextService`. Cite: `backend/app/vat/services/intake.py:62-70`.
- Page-level selected-client filters use `client_record_id` for exact `ClientRecord` matching on list, grouped, group-items, and status-summary endpoints. `client_name` is retained only as a free-text/fuzzy API filter.
- Closed or frozen clients cannot create new VAT work items. Cite: `backend/app/vat/services/intake.py:65-68`.
- Effective VAT frequency is derived from legal entity type and `vat_reporting_frequency`: `OSEK_PATUR` and `EMPLOYEE` resolve to `exempt`; otherwise the configured VAT frequency is used, falling back to `monthly`. Cite: `backend/app/vat/services/vat_type_resolver.py:6-18`.
- Exempt clients cannot create periodic VAT work items. Bi-monthly clients cannot create work items for even start months. Cite: `backend/app/vat/services/intake.py:36-48`.
- Active duplicates are blocked by service check and partial unique index on `(client_record_id, period) WHERE deleted_at IS NULL`. Cite: `backend/app/vat/services/intake.py:72-80`, `backend/app/vat/models/vat_work_item.py:134-142`.
- Creating a work item materializes or reuses a `TaxCalendarEntry`, stores its FK, and snapshots `due_date_original` and `due_date_effective` from the calendar due date. Cite: `backend/app/vat/services/intake.py:92-111`.
- Creating with `mark_pending=True` requires `pending_materials_note`; otherwise initial status is `material_received`. Cite: `backend/app/vat/services/intake.py:82-90`.
- `materials-complete` only moves `pending_materials` to `material_received`. Cite: `backend/app/vat/services/intake.py:125-155`.
- Filed work items are immutable for invoice create/update/delete. Cite: `backend/app/vat/services/data_entry_common.py:32-35`, `backend/app/vat/services/data_entry_invoices.py:68-72`, `backend/app/vat/services/data_entry_invoice_update.py:55-60`, `backend/app/vat/services/data_entry_invoice_delete.py:35-40`.
- Adding the first invoice from `material_received` auto-transitions the item to `data_entry_in_progress`. Adding invoices is otherwise allowed only in `data_entry_in_progress` or `ready_for_review`. Cite: `backend/app/vat/services/data_entry_invoices.py:141-160`.
- Invoice gross amount must be positive; VAT is split from gross using the annual configured VAT rate, with `exempt` and `zero_rate` producing zero VAT. Cite: `backend/app/vat/services/data_entry_invoices.py:96-100`, `backend/app/vat/services/vat_amounts.py:24-39`.
- Expense invoices require `expense_category`; expense tax invoices require `counterparty_id`; negative VAT and non-positive net amounts are rejected. Cite: `backend/app/vat/services/data_entry_common.py:74-110`.
- `business_activity_id` on invoice create must belong to the same legal entity as the work item's client record. Cite: `backend/app/vat/services/data_entry_invoices.py:88-94`.
- OSEK PATUR income invoices are checked against the annual ceiling; crossing the warning threshold returns a non-blocking warning, crossing the ceiling raises an error. Cite: `backend/app/vat/services/data_entry_common.py:113-138`.
- Credit notes are stored with positive amounts but contribute as negative values during aggregation. Cite: `backend/app/vat/repositories/vat_invoice_aggregation_repository.py:18-27`.
- Work-item VAT totals are recalculated after invoice create/update/delete. Output VAT includes standard-rate income VAT; input VAT is expense VAT multiplied by `deduction_rate`; `net_vat = output - input`. Cite: `backend/app/vat/services/data_entry_common.py:51-60`, `backend/app/vat/repositories/vat_invoice_aggregation_repository.py:29-87`, `backend/app/vat/repositories/vat_work_item_write_repository.py:151-171`.
- `ready-for-review` only accepts `data_entry_in_progress`; `send-back` requires a non-empty correction note and transitions back to `data_entry_in_progress`. Cite: `backend/app/vat/services/data_entry_status.py:17-49`, `backend/app/vat/services/data_entry_status.py:52-90`.
- Filing requires transition to be allowed by `VALID_TRANSITIONS`, permits optional override only with justification, and writes final filing fields plus audit entries. Cite: `backend/app/vat/services/constants.py:7-25`, `backend/app/vat/services/filing.py:46-114`.
- Filing requires `assigned_to` to be non-null; filing an unassigned item raises `VAT.ASSIGNEE_REQUIRED`. Cite: `backend/app/vat/services/filing.py:66-67`.
- Generic work-item PATCH is limited to operational metadata: `assigned_to` and `pending_materials_note`. It uses partial-update semantics, so omitted fields are not changed and explicit `null` clears nullable metadata. It does not update status, period, client identity, VAT totals, filing fields, amendment fields, or calendar snapshot fields.
- Filed work items reject generic metadata PATCH and DELETE with `VAT.FILED_IMMUTABLE`; filed VAT periods are records of filing and must not be hidden through delete.
- Work-item DELETE is soft delete only for non-filed mistaken obligations: it sets `deleted_at`, `deleted_by`, and `updated_at`, preserves invoices and audit logs, and appends a VAT audit entry. Soft-deleted items are excluded from list, lookup, detail, and client-summary query results through existing `deleted_at IS NULL` repository filters.
- `is_amendment=True` requires `amends_item_id`; omitting it raises `VAT.AMENDMENT_ID_REQUIRED`. Amendment validation (client match, filed status, cycle detection) runs only when `amends_item_id` is provided. Cite: `backend/app/vat/services/filing.py:69-73`.
- Amendments can point to an existing filed work item for the same client, and amendment cycles are rejected. Cite: `backend/app/vat/services/filing.py:24-44`.
- `business_activity_id` on invoice update must belong to the same legal entity as the work item's client record; mismatched or non-existent IDs return `BUSINESS_ACTIVITY.NOT_FOUND` (404, no existence leak). Cite: `backend/app/vat/services/data_entry_invoice_update.py:79-87`.
- `due_date_original` is immutable after it is first set. If `due_date_effective` differs from `due_date_original`, a non-empty `due_date_override_reason` is required. Cite: `backend/app/vat/models/due_date_snapshot_events.py:9-46`.
- Audit actions are stored as strings from service constants, not enum values. Cite: `backend/app/vat/models/vat_audit_log.py:3-12`, `backend/app/vat/services/constants.py:27-35`.

## Error codes

The `VAT.REASON` codes this domain raises. Registry: `docs/backend/error-codes.md`.

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
| `VAT.FILED_IMMUTABLE` | 400 via `AppError` | Invoice mutation, generic metadata update, or delete attempted after filing |
| `VAT.NET_NOT_POSITIVE` | 400 via `AppError` | Non-positive net/gross amount |
| `VAT.NEGATIVE_VAT` | 400 via `AppError` | Negative VAT amount |
| `VAT.EXPENSE_CATEGORY_REQUIRED` | 400 via `AppError` | Expense invoice missing category |
| `VAT.COUNTERPARTY_ID_REQUIRED` | 400 via `AppError` | Expense tax invoice missing supplier ID |
| `VAT.OSEK_PATUR_CEILING_EXCEEDED` | 400 via `AppError` | OSEK PATUR annual turnover would exceed configured ceiling |
| `VAT.JUSTIFICATION_REQUIRED` | 400 via `AppError` | Missing correction note or override justification |
| `VAT.ASSIGNEE_REQUIRED` | 400 via `AppError` | Filing attempted when `assigned_to` is null |
| `VAT.AMENDMENT_ID_REQUIRED` | 400 via `AppError` | `is_amendment=True` submitted without `amends_item_id` |
| `VAT.AMENDED_ITEM_NOT_FOUND` | 404 via `AppError` | `amends_item_id` does not match any work item |
| `VAT.AMENDED_ITEM_WRONG_CLIENT` | 400 via `AppError` | Amended item belongs to a different client record |
| `VAT.AMENDED_ITEM_NOT_FILED` | 400 via `AppError` | Amended item is not in `filed` status |
| `VAT.AMENDMENT_CYCLE` | 400 via `AppError` | Amendment chain would create a cycle |
| `VAT.MISSING_FINAL_AMOUNT` | 400 via `AppError` | `net_vat` is null and no override amount supplied at filing |

Related codes raised from this module but owned by another namespace:
- `BUSINESS_ACTIVITY.WRONG_CLIENT` — invoice create: activity does not belong to the work item's legal entity. Cite: `backend/app/vat/services/data_entry_invoices.py:88-94`.
- `BUSINESS_ACTIVITY.NOT_FOUND` — invoice update: `business_activity_id` not found or belongs to a different legal entity (404, no existence leak). Cite: `backend/app/vat/services/data_entry_invoice_update.py:79-87`.

## Known issues

No open known issues.

## Resolved issues

- **F-007, F-008, F-009, F-010** (2026-06-04): Resolved.

## Decisions (preserved)

1. **VAT workflow objects are client-record scoped.** `VatWorkItem` links to `client_record_id`, not directly to `legal_entity_id`; any legal-entity join goes through `ClientRecord`. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat/models/vat_work_item.py:46-48`.
2. **VAT reporting frequency is independent from advance-payment frequency.** VAT work-item creation uses `vat_reporting_frequency` / `VatType` only; it does not derive from advance-payment settings. Confirmed in `backend/app/vat/services/vat_type_resolver.py:6-18`.
3. **Period is the business identity.** Active uniqueness is by `(client_record_id, period)`, and period is stored as `YYYY-MM`. Confirmed in `backend/app/vat/models/vat_work_item.py:52-56` and `backend/app/vat/models/vat_work_item.py:134-142`.
4. **TaxCalendarEntry is the regulatory deadline source, but work items store snapshots.** Work-item creation materializes a VAT calendar entry and snapshots original/effective due dates. Confirmed in `backend/app/vat/services/intake.py:92-111`.
5. **`due_date_effective` is the intended date for overdue/reminder logic.** The code stores and enforces effective-date snapshot integrity; no current endpoint updates it. Preserved from `backend/docs/domain_decisions_v3.md` and confirmed in `backend/app/vat/models/due_date_snapshot_events.py:9-46`.
6. **Filed VAT periods are immutable for invoice mutation.** Confirmed in `backend/app/vat/services/data_entry_common.py:32-35`.
7. **Credit notes reverse totals without storing negative document amounts.** Confirmed in `backend/app/vat/models/vat_invoice.py:79-81` and `backend/app/vat/repositories/vat_invoice_aggregation_repository.py:18-27`.
8. **Business activity is optional tagging, not the VAT owner.** `business_activity_id` may be null on invoices, and VAT work items remain owned by the client record. Confirmed in `backend/app/vat/models/vat_invoice.py:52-56`.
9. **Audit action names are flexible strings.** They are intentionally not enum-backed to avoid migrations for new actions. Confirmed in `backend/app/vat/models/vat_audit_log.py:3-12`.

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
