## Scope

This file owns only:

- Canonical current-state documentation for the invoices domain.

This file must not contain:

- Architecture/API rules (link to docs/architecture/*).
- Charge lifecycle rules beyond invoice attachment preconditions.

Source of truth: mandatory

# Invoices

The invoices domain stores external invoice references attached one-to-one to office charges. It owns the `Invoice` model, repository, service, schemas, and, since Phase 2, a small HTTP API for attaching an invoice reference to an issued charge and fetching the invoice for a charge.

Last verified against code + current OpenAPI export: 2026-06-10.

## Endpoints

All endpoints require role `ADVISOR` or `SECRETARY`.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/v1/invoices` | Attach an external invoice reference to an issued charge. |
| `GET` | `/api/v1/invoices/charge/{charge_id}` | Fetch the invoice reference attached to a charge. |

Router files: `backend/app/invoices/api/invoices.py`, `backend/app/invoices/api/routers.py`. The router is registered in `backend/app/router_registry.py`.

## Model & fields

### Invoice (`backend/app/invoices/models/invoice.py`)

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | autoincrement |
| `charge_id` | int FK -> `charges.id` | no | unique, indexed |
| `provider` | str | no | external invoice provider name |
| `external_invoice_id` | str | no | provider-side invoice id |
| `document_url` | str | yes | optional invoice document URL |
| `issued_at` | datetime | no | external invoice issue timestamp |
| `created_at` | datetime | no | created by backend default |

## Schemas

`InvoiceAttachRequest` accepts `charge_id`, `provider`, `external_invoice_id`, `issued_at`, and optional `document_url`.

`InvoiceResponse` returns `id`, `charge_id`, `provider`, `external_invoice_id`, optional `document_url`, `issued_at`, and `created_at`.

## Enums / statuses

Invoice defines no local enums. Attachment depends on `ChargeStatus.ISSUED` from the charge domain.

## Domain rules & invariants

- An invoice reference can be attached only to an existing charge.
- The target charge must be in `issued` status.
- A charge can have at most one invoice reference. This is enforced by service validation and by the unique `charge_id` column.
- Invoice metadata is immutable after creation; there is no update method or update endpoint.
- VAT report invoices are separate source-document records under `vat`; they are not backed by `app/invoices/models/invoice.py`.

## Error codes

Registry: `docs/backend/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `INVOICE.NOT_FOUND` | Charge is missing during attach, or no invoice exists for `GET /invoices/charge/{charge_id}`. |
| `INVOICE.INVALID_STATUS` | Attach requested for a charge that is not `issued`. |
| `INVOICE.CONFLICT` | Attach requested for a charge that already has an invoice. |

## Known issues

- External invoice provider integration is not implemented. `BillingService.issue_charge` does not call invoice attachment automatically.

## Decisions (preserved)

1. **Invoice linkage is downstream and manual/internal.** Charge remains the source of truth for charge lifecycle and payment state.
2. **Package renamed in Phase 7.5.** Phase 2 added the API under `app/invoice/`; Phase 7.5 renamed the package to `app/invoices/`.

## Future / planned

- Integrate invoice attachment with an external provider flow when provider integration exists.
