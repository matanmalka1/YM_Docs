## Scope

This file owns only:

- Historical invoice documentation copied from `backend/app/invoice/README.md` before canonical docs migration.

This file must not contain:

- Current source-of-truth rules.

Source of truth: historical

# Invoice Module Legacy Notes

Archived source: `backend/app/invoice/README.md`.

The invoice module managed external invoice references attached to charges: provider metadata, external invoice id, issue timestamp, and optional document URL.

Historical implementation references:

- Model: `app/invoice/models/invoice.py`
- Schemas: `app/invoice/schemas/invoice_schemas.py`
- Repository: `app/invoice/repositories/invoice_repository.py`
- Service: `app/invoice/services/invoice_service.py`

The legacy README stated that there was no standalone invoice HTTP router. That became stale in Phase 2, which added `app/invoice/api/` and registered the invoice router.

Still-preserved behavior:

- `attach_invoice_to_charge` requires an existing issued charge.
- A charge can have only one invoice reference.
- Invoice metadata is immutable after creation.
- VAT reports use a separate invoice model under `app/vat_reports/models/vat_invoice.py`.
