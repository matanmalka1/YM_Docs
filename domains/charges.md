## Scope
This file owns only:
- Canonical current-state documentation for the charges domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Charges

The charges domain owns office-managed billing charges linked to a `client_record_id`, with an optional business scope and optional annual-report linkage. It exposes a small lifecycle surface: create a draft charge, transition it through issued/paid/canceled states, soft-delete eligible charges, and list/get charges with enrichment used by the CRM UI.

Last verified against code + backend/openapi.json: 2026-06-11 for `updated_at` (#46); full-domain verification remains 2026-05-29.

## Endpoints

All paths below exist in `backend/openapi.json:6195-6675`. All routes require `ADVISOR` or `SECRETARY` via route-level `require_role(...)`, including create/mutate routes. Cite: `backend/app/charges/api/charge.py:31-36`, `50-54`, `60-64`, `70-74`, `85-88`, `115-118`, `125-128`, `150-153`.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/charges` | Create a draft charge for a client record, optionally scoped to one business |
| `GET` | `/api/v1/charges` | List non-deleted charges (thin `ChargeListItem`) with business/client/status/type/period/date filters, pagination, and per-status stats |
| `GET` | `/api/v1/charges/{charge_id}` | Fetch one non-deleted charge by id |
| `POST` | `/api/v1/charges/{charge_id}/issue` | Transition a draft charge to `issued` and stamp `issued_at` / `issued_by` |
| `POST` | `/api/v1/charges/{charge_id}/mark-paid` | Transition an issued charge to `paid` and stamp `paid_at` / `paid_by` |
| `POST` | `/api/v1/charges/{charge_id}/cancel` | Cancel a draft or issued charge and optionally store `cancellation_reason` |
| `POST` | `/api/v1/charges/bulk-action` | Apply `issue`, `mark-paid`, or `cancel` to multiple charges with partial-success semantics |
| `DELETE` | `/api/v1/charges/{charge_id}` | Soft-delete a draft or canceled charge |

## Model & fields

Model: `Charge` in `backend/app/charges/models/charge.py:31-101`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | No | autoincrement |
| `client_record_id` | int FK -> `client_records.id` | No | primary domain anchor; indexed |
| `business_id` | int FK -> `businesses.id` | Yes | optional per-business scope; indexed |
| `annual_report_id` | int FK -> `annual_reports.id` | Yes | optional link for annual-report billing |
| `charge_type` | `ChargeType` enum | No | billing category |
| `status` | `ChargeStatus` enum | No | defaults to `draft` |
| `amount` | `Numeric(10,2)` | No | always stored in ILS; no currency column |
| `period` | `String(7)` | Yes | optional `YYYY-MM` first covered month; indexed |
| `months_covered` | int | No | defaults to `1`; current schema allows `1..2` |
| `description` | text | Yes | persisted but not accepted by current create API |
| `created_at` | datetime | No | default `utcnow` |
| `updated_at` | datetime | Yes | `onupdate=utcnow`; set on real mutation (issue/pay/cancel/soft-delete); NULL until first update — never faked from `created_at` (#46) |
| `created_by` | int FK -> `users.id` | Yes | actor who created the charge |
| `issued_at` | datetime | Yes | set on issue transition |
| `issued_by` | int FK -> `users.id` | Yes | set on issue transition |
| `paid_at` | datetime | Yes | set on paid transition |
| `paid_by` | int FK -> `users.id` | Yes | set on paid transition |
| `canceled_at` | datetime | Yes | set on cancel transition |
| `canceled_by` | int FK -> `users.id` | Yes | set on cancel transition |
| `cancellation_reason` | text | Yes | optional reason captured on cancel |
| `deleted_at` | datetime | Yes | soft-delete marker |
| `deleted_by` | int FK -> `users.id` | Yes | actor who soft-deleted |

Indexes declared on the model:
- `idx_charge_client_record_period` on `client_record_id, period`. (`backend/app/charges/models/charge.py:98-100`)
- `idx_charge_status` on `status`. (`backend/app/charges/models/charge.py:100`)

API shape notes:
- Create accepts `client_record_id`, optional `business_id`, positive `amount`, enum `charge_type`, optional `period`, and `months_covered` constrained to `1..2`. (`backend/app/charges/schemas/charge.py:13-26`)
- Both read DTOs add enriched `client_name`, `business_name`, `office_client_number`, and `available_actions`. (`backend/app/charges/schemas/charge.py`, `backend/app/charges/services/charge_response_builder.py:11-20`)
- List/detail DTO split: `GET /api/v1/charges` returns the thin `ChargeListItem` (the fields the charges table renders), while detail-only fields (`description`, `annual_report_id`, audit actors `created_by`/`issued_by`/`paid_by`/`canceled_by`, `canceled_at`, `cancellation_reason`) are served only by `GET /api/v1/charges/{charge_id}` as `ChargeResponse`. The charge detail drawer fetches the charge by id. (`backend/app/charges/schemas/charge.py`, `backend/app/charges/services/charge_query_service.py`)

## Enums / statuses

### `ChargeType`

Source: `backend/app/charges/models/charge.py:15-22`.

| Value |
|-------|
| `monthly_retainer` |
| `annual_report_fee` |
| `vat_filing_fee` |
| `representation_fee` |
| `consultation_fee` |
| `other` |

### `ChargeStatus`

Source: `backend/app/charges/models/charge.py:24-28`.

| Value |
|-------|
| `draft` |
| `issued` |
| `paid` |
| `canceled` |

### Bulk action values

Source: `backend/app/charges/schemas/charge.py:81-84`.

| Value |
|-------|
| `issue` |
| `mark-paid` |
| `cancel` |

## Domain rules & invariants

- A charge is always anchored on `client_record_id`; `business_id` is optional and only valid when the business exists, is valid for create, and belongs to the same legal entity as the client record. (`backend/app/charges/services/billing_service.py:39-55`, `69-78`)
- Create rejects missing client records and inactive client records before persisting the charge. (`backend/app/charges/services/billing_service.py:69-78`)
- Amount must be positive. The request schema enforces `gt=0`, and the service re-checks `amount <= 0` and raises `CHARGE.AMOUNT_INVALID`. (`backend/app/charges/schemas/charge.py:16`, `backend/app/charges/services/billing_service.py:67-68`)
- New charges are always created in `draft` status with optional `period` and `months_covered`, and the create audit payload records only `amount` and `charge_type`. (`backend/app/charges/repositories/charge_repository.py:20-43`, `backend/app/charges/services/billing_service.py:79-97`)
- `period` must match `YYYY-MM`, and `months_covered` is currently limited to at most `2`. (`backend/app/charges/schemas/charge.py:18-26`, `backend/app/charges/services/constants.py:2-3`)
- Lifecycle transitions are strict:
  - only `draft` can move to `issued`; issuing stamps `issued_at` and `issued_by`. (`backend/app/charges/services/billing_service.py:99-123`)
  - only `issued` can move to `paid`; paying stamps `paid_at` and `paid_by`. (`backend/app/charges/services/billing_service.py:125-149`)
  - `paid` cannot be canceled; canceling an already canceled charge raises conflict; canceling stores `canceled_at`, `canceled_by`, and optional `cancellation_reason`. (`backend/app/charges/services/billing_service.py:151-182`)
- Soft delete is allowed only for `draft` or `canceled` charges. The repository base queries exclude `deleted_at IS NOT NULL`, so deleted charges disappear from get/list/count/stats/aging queries. (`backend/app/charges/services/billing_service.py:184-196`, `backend/app/charges/repositories/charge_repository.py:68-70`, `183-204`, `221-244`, `246-297`)
- List filters support `business_id`, `client_record_id`, `status`, `charge_type`, `period`, `issued_after`, `issued_before`, `page`, and `page_size`. Results are ordered by `created_at DESC`. (`backend/app/charges/api/charge.py:90-112`, `backend/app/charges/repositories/charge_repository.py:45-117`)
- Enriched read responses derive `client_name` from client-record reads, `business_name` from `BusinessRepository`, `office_client_number` from the client record, and `available_actions` from `get_charge_actions(...)`. (`backend/app/charges/services/charge_query_service.py:28-37`, `105-125`, `161-167`; `backend/app/charges/services/charge_response_builder.py:11-20`)
- Bulk actions are partial-success by design: each id is processed independently; domain `AppError`s become per-item failures with the localized message, and unexpected exceptions become a generic internal-error failure item without aborting the rest. (`backend/app/charges/services/bulk_billing_service.py:15-48`)
- Issued unpaid charges become derived work-queue items only after `issued_at` is at least `30` days old. The work-queue due date is computed as `issued_at.date() + 30 days`; no separate task row is created. (`backend/app/charges/services/constants.py:1`, `backend/app/work_queue/services/billing_items.py:58-92`)

## Error codes

Registry: `docs/backend/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `CHARGE.CLIENT_NOT_FOUND` | `_validate_charge_scope` cannot find the client record while validating scope (`backend/app/charges/services/billing_service.py:45-50`) |
| `CHARGE.CLIENT_RECORD_NOT_FOUND` | `create_charge` cannot find the client record before active/scope checks (`backend/app/charges/services/billing_service.py:69-74`) |
| `CHARGE.AMOUNT_INVALID` | Amount is zero/negative in the service layer (`backend/app/charges/services/billing_service.py:67-68`) |
| `CHARGE.NOT_FOUND` | Requested charge id does not exist for get/issue/pay/cancel/delete (`backend/app/charges/services/billing_service.py:101-102`, `127-128`, `158-159`, `186-187`, `199-201`) |
| `CHARGE.INVALID_STATUS` | Status transition or delete is not allowed from the current state (`backend/app/charges/services/billing_service.py:104-107`, `130-133`, `160-161`, `189-191`) |
| `CHARGE.CONFLICT` | Cancel is requested for an already canceled charge (`backend/app/charges/services/billing_service.py:162-163`) |

Cross-domain codes raised through called guards:
- `BUSINESS.NOT_FOUND`, `BUSINESS.CLOSED`, `BUSINESS.FROZEN` can surface from `validate_business_for_create(...)`. (`backend/app/charges/services/billing_service.py:53`)

## Known issues

No open known issues.

## Resolved issues

- **F-013** (2026-06-05): List `stats` did not respect most list filters. Fixed: `stats_by_status()` now accepts and applies `business_id`, `period`, `issued_after`, and `issued_before` in addition to `client_record_id` and `charge_type`. `list_charges_paginated()` threads all active filters through.
- **F-014** (2026-06-05): OpenAPI advertised `X-Idempotency-Key` as optional on bulk action, but runtime rejected missing keys. Fixed: `require_idempotency_key` header parameter changed from `str | None` with `default=None` to required `str`. FastAPI now generates `required: true` and returns 422 for missing header.

## Decisions (preserved)

Still-true decisions preserved from shared legacy docs and verified against current code:

1. **Charges are office-managed internal billing records, not external invoice-provider events.** `BillingService.issue_charge()` only changes internal status/audit state; it does not call an external billing provider. (`backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md:95-108`, `backend/app/charges/services/billing_service.py:99-123`)
2. **Invoice linkage is downstream and manual/internal.** The charge domain owns the charge lifecycle; invoice attachment is a separate concern and the charge model only exposes a nullable one-to-one `invoice` relationship. (`backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md:97-102`, `backend/app/charges/models/charge.py:94-96`)
3. **Unpaid issued charges surface in the work queue as derived system items after the threshold passes.** Business scoping applies only to charge items; VAT, annual reports, and advance payments remain client-level in that queue view. (`backend/docs/backend/domains/work-queue.md:47-52`, `68-72`; `backend/app/work_queue/services/billing_items.py:58-92`)

## Future / planned

- A generic audit-history UI may later be shared across `charge` and other audited entities, but that UI reuse is not implemented in the charge domain itself. (`backend/docs/history-vs-timeline.md:24-29`)

## Historical notes

No `backend/docs` file exists that is dedicated to the charge domain, so this docs-only pass did not create `docs/archive/charge-legacy.md` and did not pointer-rewrite any legacy domain file. Historical rationale that still mentions charge remains in shared files such as `backend/docs/domain_decisions_v3.md`, `backend/docs/backend/domains/work-queue.md`, `backend/docs/domain_model/DOMAIN_MODEL_REVIEW_SUMMARY.md`, and `backend/docs/history-vs-timeline.md`, which were left untouched to avoid cross-domain conflicts.
