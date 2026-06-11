## Scope

This file owns only:

- Conventions for `*UpdateRequest` (PATCH) schemas: empty-PATCH policy, unknown-field policy, partial-update / explicit-null semantics.
- Documented exceptions: single-payload required-field update schemas.
- The `ClientUpdateRequest` field-ownership decision (issue 40).

This file must not contain:

- Full API contract rules (see `docs/architecture/api-contracts.md`).
- Per-domain endpoint behavior.

Source of truth: reference (defers to `docs/architecture/api-contracts.md` for binding contract rules).

# Update Request (PATCH) Conventions

## Shared base: `NonEmptyUpdateMixin`

All `*UpdateRequest` schemas inherit `app/core/schemas/validation.py::NonEmptyUpdateMixin`, which:

- Sets `model_config = ConfigDict(extra="forbid")` — **unknown request fields are rejected with 422**, never silently ignored. Removing or renaming a field becomes visible contract drift instead of being swallowed.
- Rejects an empty `{}` PATCH body with **422** (checked via `model_fields_set`, so an explicit `null` still counts as a provided field).

### Empty PATCH returns 422 (not 400)

An empty PATCH is rejected at the schema-validation layer by Pydantic/FastAPI, which yields **422**. This intentionally differs from a 400 suggestion: doing it in the schema avoids duplicating the check in every endpoint. 422 and 400 are not the same status; 422 is the deliberate choice.

## Partial PATCH semantics

- Update endpoints apply changes with `request.model_dump(exclude_unset=True)`. Never `exclude_none` — `exclude_none` cannot distinguish "omitted" from "explicitly set to null" and silently drops legitimate null-clears.
- A field that is **nullable in the database/domain** may be cleared by sending an explicit `null`.
- A field that is **non-nullable in the model** (including columns with a non-null default such as `Business.status`, `Task.priority`, `ExpenseLine.recognition_rate`) must **reject explicit null with 422**. This is enforced by a per-schema `model_validator(mode="after")` that checks `field in self.model_fields_set and getattr(self, field) is None`.

> Classification rule: clearability is determined by the **model column nullability**, not by whether the field is optional in the Create schema. Create-optional with a default ≠ clearable.

## Business-required text fields

Fields that are business identifiers/titles use `NonBlankStr` (`app/core/api_types.py`), which strips whitespace then enforces `min_length=1`, so both `""` and `"   "` fail with 422. Applied to: `CorrespondenceUpdateRequest.subject`, `BusinessUpdateRequest.business_name` (with `max_length=100` preserved), `AuthorityContactUpdateRequest.name`, `ClientUpdateRequest.full_name`, `EntityNoteUpdateRequest.note`, `UserUpdateRequest.full_name`, `TaskUpdateRequest.title` (with `max_length=500` preserved). Free-form text (`notes`, `description`, `internal_notes`, `amendment_reason`, `custom_deadline_note`) is **not** constrained this way.

## Documented exceptions: single-payload required-field updates

Two update schemas intentionally keep a **required** field, because that field is the entire payload of the update — omitting it would be a meaningless no-op:

| Schema | Required field | Reason |
|---|---|---|
| `EntityNoteUpdateRequest` | `note` (`NonBlankStr`) | A note update with no note text is meaningless. |
| `AnnexDataUpdateRequest` | `data` (`dict`) | The annex line update *is* the data payload. |

These still inherit `NonEmptyUpdateMixin` (so `{}` → 422, unknown field → 422). They are exempt only from the "all fields optional" partial-PATCH shape.

`DeadlineUpdateRequest` is **not** an exception: it has two fields (`deadline_type`, `custom_deadline_note`), so `deadline_type` was made optional (`POST /annual-reports/{report_id}/deadline` keeps the existing type when `deadline_type` is omitted). Explicit `null` on `deadline_type` is rejected (non-nullable column).

## VAT invoice update (`VatInvoiceUpdateRequest`)

The VAT invoice PATCH (`PATCH /vat/work-items/{item_id}/invoices/{invoice_id}`) follows the full partial-PATCH contract: the endpoint forwards `model_dump(exclude_unset=True)`, and the service (`data_entry_invoice_update.update_invoice`) applies only the keys present in the patch (presence-based, not None-based). Net/VAT are recomputed only when `gross_amount` or `rate_type` is sent; `deduction_rate` only when `expense_category` is sent.

- **Reject explicit null** (non-nullable columns): `gross_amount`, `invoice_number`, `invoice_date`, `counterparty_name`, `rate_type`.
- **Reject explicit null** `expense_category`: although the column is nullable, `deduction_rate` (non-nullable) is derived from the category and cannot be safely reset, so clearing the category is disallowed.
- **Null clears** (nullable, no derived dependency): `business_activity_id`, `counterparty_id`, `counterparty_id_type`, `document_type`.

# `ClientUpdateRequest` field ownership (issue 40)

| Field | In Create? | In Update? | Owner | Editable from UI? | Reason |
|---|---|---|---|---|---|
| `status` | No | Yes | frontend/user | Yes | Lifecycle status edited via the client edit form; maps to non-nullable `ClientRecord.status`, so explicit null is rejected. |
| `annual_revenue` | No | Yes | frontend/user | Yes | Business metadata entered manually in the edit form; nullable. |
| `advance_rate_updated_at` | No | **No (removed)** | backend/server | No | Server-owned timestamp. `ClientUpdateService` stamps it with `israel_today()` only when `advance_rate` actually changes (diff against the existing record). Removed from `ClientUpdateRequest`; with `extra="forbid"`, a client that still sends it gets a 422 unknown-field error. Remains a read-only field on `ClientRecordResponse`. |
