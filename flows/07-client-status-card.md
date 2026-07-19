## Scope

This file owns only:
- Documentation of the client status card aggregation as it exists in the code.

Source of truth: reference

# Flow: Client Status Card

## 1. Trigger

`GET /clients/{client_record_id}/status-card`

## 2. Entry Point in Code

```
backend/app/businesses/services/status_card_service.py
  StatusCardService.get_status_card(client_id, year=None)
```

Router: `backend/app/businesses/api/client_status_card_router.py`

## 3. Sequence

1. `ClientRecordRepository.get_by_id(client_id)` — verify client exists; raise 404 if not.
2. `resolved_year = year or utcnow().year`.
3. Seven independent sequential queries (no batching between them):

   **a. VAT summary card** (`_vat_card`):
   - `VatWorkItemRepository.list_by_client_record(client_record_id)` — all non-deleted VAT work items.
   - Filter in Python: only items where `period.startswith(f"{resolved_year}-")`.
   - Compute: `net_vat_total` (sum of `net_vat` for year), `periods_filed` (count with status FILED), `periods_total` (total count for year), `latest_period` (max period string for year).

   **b. Annual report card** (`_annual_report_card`):
   - `AnnualReportRepository.get_by_client_record_year(client_record_id, resolved_year)` — single row.
   - Returns: status, form_type, filing_deadline (`ApiDateTime | None`), refund_due, tax_due.
   - If no report: returns empty card.

   **c. Charges card** (`_charges_card`):
   - `ChargeRepository.list_charges_by_client_record(client_record_id, status=ISSUED, page=1, page_size=10_000)`.
   - Not year-filtered.
   - Compute: `total_outstanding` (sum of amount), `unpaid_count` (len of results).

   **d. Advance payments card** (`_advance_payments_card`):
   - `AdvancePaymentRepository.list_by_client_record_year(client_record_id, year, page=1, page_size=10_000)`.
   - Compute: `total_paid` (sum of paid_amount), `count`.

   **e. Binders card** (`_binders_card`):
   - `BinderRepository.list_by_client_record(client_record_id)` — all binders.
   - In Python: `active = [b for b in rows if b.location_status != HANDED_OVER]`.
   - Compute: `active_count`, `in_office_count` (where `location_status == IN_OFFICE`).

   **f. Documents card** (`_documents_card`):
   - `PermanentDocumentRepository.list_by_client_record(client_record_id)`.
   - Compute: `total_count`, `present_count` (where `is_present == True`).

   **g. Tasks card** (`_tasks_card`):
   - `TaskRepository.list_by_client_id(...)` scoped to `status=OPEN`.
   - Compute: `open_count` for non-deleted tasks explicitly related to the client.

4. Return `ClientStatusCardResponse` with all 7 cards.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `businesses` | Owns the endpoint and service |
| `clients` | ClientRecord guard |
| `vat` | VatWorkItem read |
| `annual_reports` | AnnualReport read |
| `charge` | Charge read |
| `advance_payments` | AdvancePayment read |
| `binders` | Binder read |
| `permanent_documents` | PermanentDocument read |
| `tasks` | Open client-task count |

## 5. Side Effects

None. Read-only.

## 6. Transaction Boundaries

Read-only. All queries run in the same DB session. No writes.

## 7. Idempotency

N/A — read-only.

## 8. Locks / Concurrency

None.

## 9. Preconditions

- Client must exist (ClientRecord).

## 10. Blockers / Validation Failures

| Condition | Error Code | HTTP |
|-----------|-----------|------|
| Client not found | `CLIENT_RECORD.NOT_FOUND` | 404 |

## 11. Derived State

All card values are computed from live DB state. Nothing is cached or stored.

| Field | Source |
|-------|--------|
| `net_vat_total` | Sum of VatWorkItem.net_vat for the year |
| `periods_filed` | Count of VatWorkItems with status FILED for the year |
| `total_outstanding` (charges) | Sum of all ISSUED Charge.amount for the client (all years) |
| `in_office_count` | Binders where location_status == IN_OFFICE |
| `open_count` (tasks) | Count of non-deleted client tasks with status OPEN |

## 12. N+1 Risk

Low: 7 flat queries per request.

**Potential issue**: `_charges_card` uses `page_size=10_000`. For clients with many charges, this loads all rows into memory. No pagination or aggregate-only query.

**Potential issue**: `_vat_card` loads all VAT work items for the client, then filters by year in Python. If a client has many years of data, unnecessary rows are loaded.

## 13. Tests

- `tests/businesses/api/test_business_binders_api.py` (partial coverage)
- `tests/businesses/service/test_business_status_card_service.py` (open client-task aggregation)

## 14. Documentation Target

- `docs/domains/businesses.md` — status card endpoint ownership
- `docs/domains/clients.md` — cross-domain status summary
