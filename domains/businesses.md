## Scope
This file owns only:
- Canonical current-state documentation for the businesses domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Businesses

The businesses domain manages operational business activities that belong to a client. A single client (legal entity) may hold multiple businesses representing distinct operational activities. The legal and tax identity is anchored on the `LegalEntity`; `Business` represents an activity under that entity. This domain also owns the client status-card aggregation endpoint, which summarises cross-domain state (VAT, annual report, charges, advance payments, binders, documents) for a given client and year.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All paths confirmed in `backend/openapi.json`.

| Method | Path | Roles | Purpose |
|--------|------|-------|---------|
| `POST` | `/api/v1/clients/{client_record_id}/businesses` | ADVISOR | Create a new business under a client |
| `GET` | `/api/v1/clients/{client_record_id}/businesses` | ADVISOR, SECRETARY | List businesses for a client (paginated) |
| `GET` | `/api/v1/clients/{client_record_id}/businesses/{business_id}` | ADVISOR, SECRETARY | Get a single business |
| `PATCH` | `/api/v1/clients/{client_record_id}/businesses/{business_id}` | ADVISOR, SECRETARY | Partial update of a business |
| `DELETE` | `/api/v1/clients/{client_record_id}/businesses/{business_id}` | ADVISOR | Soft-delete a business (204) |
| `POST` | `/api/v1/clients/{client_record_id}/businesses/{business_id}/restore` | ADVISOR | Restore a soft-deleted business |
| `GET` | `/api/v1/clients/{client_record_id}/status-card` | ADVISOR, SECRETARY | Cross-domain status card for a client |

> Notes endpoints (`/businesses/{business_id}/notes`) exist in `backend/openapi.json` but are owned by the `notes` domain (`backend/app/notes/api/business_notes.py`), not this domain.

Router sources:
- `backend/app/businesses/api/client_businesses_router.py` — prefix `/clients/{client_record_id}/businesses`
- `backend/app/businesses/api/client_status_card_router.py` — prefix `/clients`
- Aggregated in `backend/app/businesses/api/routers.py`

## Model & fields

Model: `Business` in `backend/app/businesses/models/business.py`.  
Mixin: `SoftDeletableMixin` (adds `deleted_at`, `deleted_by`, `restored_at`, `restored_by`).

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | auto-increment |
| `legal_entity_id` | int FK → `legal_entities.id` | no | indexed; ownership anchor |
| `business_name` | String | no | required; max 100 chars (schema); unique per legal entity among non-deleted (partial index) |
| `status` | `BusinessStatus` enum | no | default `active` |
| `opened_at` | date | no | |
| `closed_at` | date | yes | set automatically when status → `closed` |
| `phone_override` | String(20) | yes | overrides owner-person phone for this business |
| `email_override` | String(254) | yes | overrides owner-person email for this business |
| `notes` | Text | yes | |
| `created_by` | int FK → `users.id` | yes | |
| `created_at` | datetime | no | default utcnow |
| `updated_at` | datetime | yes | on-update utcnow |

Soft-delete columns (from `SoftDeletableMixin`): `deleted_at`, `deleted_by`, `restored_at`, `restored_by`.

Computed property `full_name` (model:81): returns `business_name`, falls back to `legal_entity.official_name`.

DB indexes (`business.py:94`):
- `ix_business_status` on `status`
- `ix_business_legal_entity_name_active` — unique partial on `(legal_entity_id, business_name)` where `business_name IS NOT NULL AND deleted_at IS NULL`

Contact resolution (`backend/app/businesses/services/business_contact_service.py`):
- `contact_email`: returns `email_override` if set, else owner-person email via `PersonLegalEntityRole.OWNER` link
- `contact_phone`: same pattern for phone

## Enums / statuses

Source: `backend/app/businesses/models/business.py:22`

```
BusinessStatus(str, Enum):
    ACTIVE  = "active"
    FROZEN  = "frozen"
    CLOSED  = "closed"
```

## Domain rules & invariants

Sources: `backend/app/businesses/services/business_service.py`, `business_guards.py`, `business_lifecycle_service.py`.

**Create:**
- Client (`client_record_id`) must exist; missing → `CLIENT.NOT_FOUND` (`business_service.py:62`).
- If the legal entity already has at least one non-deleted business and **all** of them are `closed`, creating a new business is blocked → `BUSINESS.CLIENT_ALL_CLOSED` (`business_service.py:67–71`). This prevents silently re-opening a fully-closed client.
- `business_name` must be unique (case-insensitive, trimmed) among existing non-deleted businesses of the same legal entity → `BUSINESS.NAME_CONFLICT` (`business_service.py:73–84`). Also enforced at DB level by the partial unique index.
- Generic DB integrity error on create → `BUSINESS.CONFLICT` (`business_service.py:94–98`).
- `opened_at` is optional in the request; service defaults to `date.today()` (`business_service.py:64`).

**Update (`PATCH`):**
- Setting `status` to `frozen` or `closed` requires `ADVISOR` role → `BUSINESS.FORBIDDEN` (`business_service.py:165–166`).
- Setting `status` to `closed` auto-sets `closed_at` to today if not provided (`business_service.py:168`).
- Setting `status` to `active` clears `closed_at` (`business_service.py:170`).
- Unknown/invalid status value → `BUSINESS.INVALID_STATUS` (`business_service.py:157–161`).

**Delete:**
- Soft-delete only (`business_lifecycle_service.py:22`). Sets `deleted_at` + `deleted_by`, writes audit record.

**Restore:**
- Only `ADVISOR` may restore → `BUSINESS.FORBIDDEN` (`business_lifecycle_service.py:27`).
- Business must exist (including deleted) → `BUSINESS.NOT_FOUND`.
- Business must be soft-deleted; if not → `BUSINESS.NOT_DELETED` (`business_lifecycle_service.py:32`).
- Restore sets `status` back to `active` and clears `deleted_at` (`business_repository.py:62–63`).

**Ownership guard:**
- Every route validates that the requested `business_id` belongs to the `legal_entity_id` of the given `client_id`; mismatch raises `BUSINESS.NOT_FOUND` (`business_guards.py:37–43`).

**Guard helpers** (used by downstream domains on their own create flows):
- `get_business_or_raise(db, business_id)` — fetch or raise `BUSINESS.NOT_FOUND` (`business_guards.py:8`)
- `assert_business_allows_create(business)` — raises `BUSINESS.CLOSED` or `BUSINESS.FROZEN` if status blocks new records (`business_guards.py:16–27`)
- `validate_business_for_create(db, business_id)` — combines both (`business_guards.py:30–33`)

**Operational signals** (`backend/app/businesses/services/signals_service.py`):
- `compute_business_operational_signals` — dynamically computed, not persisted. Returns `missing_documents` list. Advisory/UX only, non-blocking.

**Status card** (`backend/app/businesses/services/status_card_service.py`):
- Aggregates for a given `client_id` and `year` (defaults to current year):
  - VAT: net total, filed count, total count, latest period — from `VatWorkItem` filtered by year prefix
  - Annual report: status, form_type, filing_deadline, refund_due, tax_due
  - Charges: total outstanding (ISSUED only), unpaid count
  - Advance payments: total paid, count — filtered by year
  - Binders: active count (non HANDED_OVER), in-office count
  - Documents: total count, present count

**All endpoints require minimum `ADVISOR` or `SECRETARY` role** (set on router-level dependency).

**Audit:** create, update, delete, and restore operations write to the audit log via `EntityAuditWriter` with entity type `ENTITY_BUSINESS` (`audit/constants.py`). Source: `business_service.py`, `business_lifecycle_service.py`.

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | HTTP | Raised by |
|------|------|-----------|
| `BUSINESS.NOT_FOUND` | 404 | business missing or not owned by route's client |
| `BUSINESS.NAME_CONFLICT` | 409 | duplicate active business_name per legal entity |
| `BUSINESS.CONFLICT` | 409 | generic DB integrity error on create |
| `BUSINESS.CLIENT_ALL_CLOSED` | 409 | all existing businesses for client are closed |
| `BUSINESS.NOT_DELETED` | 409 | restore attempted on non-deleted business |
| `BUSINESS.FORBIDDEN` | 403 | non-ADVISOR attempt to freeze/close/restore |
| `BUSINESS.CLOSED` | 403 | downstream guard: record creation on closed business |
| `BUSINESS.FROZEN` | 403 | downstream guard: record creation on frozen business |
| `BUSINESS.INVALID_STATUS` | 400 | unrecognised status value on update |
| `CLIENT.NOT_FOUND` | 404 | client_id not found during create/update |
| `CLIENT_RECORD.NOT_FOUND` | 404 | raised by status-card service |

Error envelope follows global format (`app/core/exceptions.py`): `error.code`, `error.message`, `error.details`, `error.request_id`.
See `docs/architecture/api-contracts.md` for envelope spec.

## Known issues

No open known issues.

## Resolved issues

- **F-006** (2026-06-04): `constants.py` imported `EntityType` from `models/business.py`, which did not define it. Fixed: the module imports `EntityType` from `app.common.enums`.
- **Businesses-001** (2026-06-05): `backend/app/businesses/README.md` listed `business_repository_read.py` and `business_update_service.py` as source files, but neither existed. README has since been rewritten as a pointer-only file — stale references are gone.

## Decisions (preserved)

From `backend/docs/domain_decisions_v3.md` (Decision #1 entity architecture, locked):

- **Business anchors on `legal_entity_id`, not `client_record_id`.** Workflow objects (VAT, advances, annual report) anchor on `client_record_id`. Business is a separate branch representing an operational activity on the legal entity, not an office-managed workflow object.
- **Status lives only on the business object** (Decision #5, locked). No separate status table.
- **Workflow objects are not client-scoped through Business.** The join path to `LegalEntity` always goes through `ClientRecord`, never directly from a workflow object through `Business`.

## Future / planned

None identified from legacy docs. The README's `has_conflicting_sole_trader` stub in `business_repository.py:99–105` always returns `False` — sole-trader conflict detection is not yet implemented. Treat as a placeholder, not a live rule.

## Historical notes

No legacy `backend/docs` file exists that is specific to the businesses domain.  
The `backend/app/businesses/README.md` is the in-module reference and has been superseded by this document as the canonical domain doc.  
Relevant cross-domain architectural decisions are preserved in `backend/docs/domain_decisions_v3.md`.
