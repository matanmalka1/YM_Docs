## Scope
This file owns only:
- Canonical current-state documentation for the permanent-documents domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Permanent Documents

The permanent-documents domain stores durable client and business files — identity documents, engagement agreements, tax forms, bank approvals, withholding certificates, and similar records. Each document is versioned: uploading a new file for the same `(client_record_id, business_id, document_type, tax_year)` combination supersedes the previous version rather than replacing it. Documents are soft-deleted; the storage object is not removed. Operational signals report missing required document types per client.

Last verified against code + backend/openapi.json: 2026-06-05.

## Endpoints

All paths confirmed in `backend/openapi.json`. Router prefix `/documents` is mounted under `/api/v1` via `app/router_registry.py`.

| Method | Path | Roles | Purpose |
|--------|------|-------|---------|
| POST | /api/v1/documents/upload | ADVISOR, SECRETARY | Upload a new (or next-version) document (multipart form) |
| GET | /api/v1/documents/client/{client_record_id} | ADVISOR, SECRETARY | List active documents for a client; optional `?tax_year=` filter |
| GET | /api/v1/documents/client/{client_record_id}/signals | ADVISOR, SECRETARY | Advisory: return missing required document types |
| GET | /api/v1/documents/client/{client_record_id}/versions | ADVISOR, SECRETARY | Version history for one `document_type`; required `?document_type=`, optional `?tax_year=` |
| GET | /api/v1/documents/client/{client_record_id}/{document_id}/download-url | ADVISOR, SECRETARY | Presigned download URL (expires 1 hour); ownership verified against `client_record_id` |
| GET | /api/v1/documents/annual-report/{report_id} | ADVISOR, SECRETARY | List documents linked to an annual report |
| DELETE | /api/v1/documents/client/{client_record_id}/{document_id} | ADVISOR | Soft-delete a document; ownership verified against `client_record_id` |
| PUT | /api/v1/documents/client/{client_record_id}/{document_id}/replace | ADVISOR | Replace file in-place (same record, incremented version); ownership verified |

## Model & fields

`PermanentDocument` — table `permanent_documents`. Source: `backend/app/documents/permanent_documents/models/permanent_document.py`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | autoincrement |
| client_record_id | int FK → client_records.id | no | denormalized for fast queries without JOIN; indexed |
| business_id | int FK → businesses.id | yes | NULL for CLIENT-scoped documents; indexed |
| scope | DocumentScope enum | no | derived from whether `business_id` is provided |
| document_type | DocumentType enum | no | |
| storage_key | str | no | S3/R2 object key |
| original_filename | str | yes | |
| file_size_bytes | bigint | yes | |
| mime_type | str | yes | |
| tax_year | smallint | yes | indexed |
| is_present | bool | no | default `true` |
| is_deleted | bool | no | default `false` (soft delete) |
| status | DocumentStatus enum | no | default `pending` |
| version | int | no | default `1` |
| superseded_by | int FK → permanent_documents.id | yes | self-FK; only the latest version has this NULL |
| annual_report_id | int FK → annual_reports.id | yes | links supporting docs to a specific report |
| uploaded_by | int FK → users.id | no | |
| uploaded_at | datetime | no | UTC |
| approved_by | int FK → users.id | yes | |
| approved_at | datetime | yes | |
| rejected_by | int FK → users.id | yes | |
| rejected_at | datetime | yes | |

DB-level `CheckConstraint` at `models/permanent_document.py:130`:
```
(scope = 'client') OR (scope = 'business' AND business_id IS NOT NULL)
```

Composite indexes (`models/permanent_document.py:134`):
- `ix_permanent_documents_client_record_type_year` on `(client_record_id, document_type, tax_year)`
- `ix_permanent_documents_business` on `(business_id, document_type, tax_year)`

## Enums / statuses

Source: `backend/app/documents/permanent_documents/models/permanent_document.py`.

**DocumentType** (line 42):
| Value | Hebrew label |
|-------|-------------|
| `id_copy` | צילום ת.ז. |
| `power_of_attorney` | ייפוי כוח |
| `engagement_agreement` | הסכם התקשרות |
| `tax_form` | טופס מס |
| `receipt` | קבלה |
| `invoice_doc` | חשבונית |
| `bank_approval` | אישור בנקאי |
| `withholding_certificate` | אישור ניכוי מס במקור |
| `nii_approval` | אישור ביטוח לאומי |
| `other` | אחר |

**DocumentStatus** (line 55):
| Value | Meaning |
|-------|---------|
| `pending` | Uploaded, awaiting review |
| `received` | Physically received |
| `approved` | Approved |
| `rejected` | Rejected — must be re-uploaded |

**DocumentScope** (line 62):
| Value | Meaning |
|-------|---------|
| `client` | Belongs to person; `business_id` is NULL |
| `business` | Belongs to specific business; `business_id` required |

**CLIENT_SCOPE_TYPES** (line 68) — `{id_copy, power_of_attorney, engagement_agreement}`. These types always belong to the person; uploading any of them with a `business_id` is rejected with `PERMANENT_DOCUMENTS.CLIENT_SCOPE_VIOLATION` (422).

## Domain rules & invariants

Source: `backend/app/documents/permanent_documents/services/permanent_document_service.py`.

- **Client existence required.** `upload_document` calls `get_client_or_raise` before any other logic (`service.py:108`).
- **Business ownership check.** When `business_id` is provided, `assert_business_belongs_to_legal_entity` verifies the business belongs to the client's legal entity (`service.py:121`). Uses `legal_entity_id` from the client record by default.
- **Scope derivation.** `scope = BUSINESS` if `business_id is not None`, else `scope = CLIENT` (`service.py:126`). `CLIENT_SCOPE_TYPES` (`id_copy`, `power_of_attorney`, `engagement_agreement`) are enforced: uploading any of these with a `business_id` raises `PERMANENT_DOCUMENTS.CLIENT_SCOPE_VIOLATION` (422).
- **File validation.** MIME type must be in `ALLOWED_MIME_TYPES`; size must be ≤ `MAX_FILE_SIZE_BYTES` (`service.py:131-137`). Checked before storage upload.
- **Versioning is atomic.** DB record flushed first; storage upload second. On storage failure, transaction rolls back (`service.py:177`). `superseded_by` set in the same final commit (`service.py:181`). Concurrent collisions raise `DOCUMENT.VERSION_CONFLICT` (409) (`service.py:186`).
- **Upload auto-approves.** New uploads are created with `status=APPROVED` and `approved_by=uploaded_by` (`service.py:167-173`). No separate approve/reject HTTP endpoints exist.
- **Storage key pattern** (`service.py:77-93`):
  - business-scoped: `businesses/{business_id}/{document_type}/{tax_year_or_permanent}/v{version}_{filename}`
  - client-scoped: `clients/{client_record_id}/{document_type}/{tax_year_or_permanent}/v{version}_{filename}`
  - When `tax_year` is None, the segment is `permanent`.
- **Soft delete.** `delete_document(client_record_id, document_id)` sets `is_deleted=True` only; storage object is not removed. Fetches via `get_by_id_and_client_record` — raises `PERMANENT_DOCUMENTS.NOT_FOUND` if the document does not exist or does not belong to the given client. All list/get queries filter `is_deleted == False`.
- **Replace in-place.** `replace_document(client_record_id, document_id, ...)` increments `version`, uploads new file, updates all file metadata, and commits — does not create a new `PermanentDocument` row or update `superseded_by`. Same ownership check as delete.
- **Default required types for signals.** `_DEFAULT_REQUIRED_TYPES = [id_copy, power_of_attorney, engagement_agreement]` (`service.py:43`). `get_client_operational_signals` returns the subset of these not yet present for the client (`service.py:249-255`).
- **List endpoints.** By default exclude soft-deleted documents and superseded versions (`superseded_by IS NULL`).

**Upload constraints** (`backend/app/documents/permanent_documents/services/constants.py`):
- `MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024` (10 MB)
- `ALLOWED_MIME_TYPES`: `application/pdf`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `image/jpeg`, `image/png`

## Error codes

Registry: `docs/backend/error-codes.md`.

| Code | HTTP | Raised when |
|------|------|-------------|
| `CLIENT_RECORD.NOT_FOUND` | 404 | Client record not found during upload or list |
| `PERMANENT_DOCUMENTS.BUSINESS_NOT_FOUND` | 404 | Business not found during business-scoped upload |
| `PERMANENT_DOCUMENTS.CLIENT_SCOPE_VIOLATION` | 422 | `id_copy`, `power_of_attorney`, or `engagement_agreement` uploaded with a `business_id` |
| `PERMANENT_DOCUMENTS.CLIENT_RECORD_NOT_FOUND` | 404 | Client record not found when computing missing docs by business |
| `PERMANENT_DOCUMENTS.NOT_FOUND` | 404 | Document not found or soft-deleted (get-download-url, delete, replace) |
| `DOCUMENT.INVALID_FILE_TYPE` | 422 | MIME type not in allowed set |
| `DOCUMENT.FILE_TOO_LARGE` | 422 | File exceeds 10 MB |
| `DOCUMENT.UPLOAD_FAILED` | 500 | Storage upload failure |
| `DOCUMENT.VERSION_CONFLICT` | 409 | Concurrent upload of same version |

## Known issues

No open known issues.

## Decisions (preserved)

From `backend/app/documents/permanent_documents/README.md` and model docstring (still true per code):

1. **Denormalized `client_record_id`.** Every document carries `client_record_id` even when `business_id` is also set, enabling fast client-level queries without a JOIN to `businesses`.
2. **Self-FK versioning.** `superseded_by` chains versions; the head (latest) always has `superseded_by = NULL`. This allows full version history without a separate versions table.
3. **DB-level scope constraint.** `CheckConstraint` enforces `business_id IS NOT NULL` for `scope=BUSINESS` at the database level, not only in application code.
4. **DB record before storage upload.** The DB row is flushed before the storage write so that a storage failure can be cleanly rolled back. The `superseded_by` pointer is committed in the same transaction as the new row.
5. **Soft delete without storage cleanup.** `is_deleted=True` marks a document deleted; the storage object is retained. Service-layer storage cleanup is a separate responsibility (not yet implemented — see Future).
6. **`rejected_by` mirrors `approved_by`.** Symmetric fields for audit completeness, even though no HTTP endpoint currently sets `rejected_by`.

## Future / planned

These behaviors are documented in the model or README but are **not implemented** in current code:

- **Standalone approve / reject endpoints.** The `DocumentStatus` enum includes `received`, `approved`, `rejected`, and the model has `approved_by / approved_at / rejected_by / rejected_at` fields, but no HTTP endpoints exist to transition status post-upload. Currently every upload is auto-approved at upload time.
- **Storage cleanup on soft delete.** Model docstring states "service layer handles storage cleanup separately." No such cleanup is implemented.
- **`is_present` flag semantics.** The field exists and defaults to `true`; `replace_document` resets it to `true`. There is no code path that sets it to `false`. The intended use (tracking physical receipt separately from upload) is not yet implemented.

## Historical notes

No legacy `backend/docs` files exist for this domain. The module README at `backend/app/documents/permanent_documents/README.md` is pointer-only; this canonical doc owns the current domain detail.
