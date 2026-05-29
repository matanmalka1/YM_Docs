## Scope
This file owns only:
- Canonical current-state documentation for the correspondence domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Correspondence

The correspondence domain manages the per-client-record log of all interactions between the tax advisory firm, its clients, and external authority contacts (e.g., Israeli Tax Authority, National Insurance). Each entry records a single interaction event (call, letter, email, meeting, or fax) anchored to a `client_record_id`, with optional scoping to a specific business and optional linkage to an authority contact. The log is the audit trail used for ITA disputes and client timeline views.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| GET | `/api/v1/clients/{client_record_id}/correspondence` | List all entries for a client (paginated, filterable) | ADVISOR, SECRETARY |
| POST | `/api/v1/clients/{client_record_id}/correspondence` | Create a new correspondence entry | ADVISOR, SECRETARY |
| GET | `/api/v1/clients/{client_record_id}/correspondence/{correspondence_id}` | Get a single entry | ADVISOR, SECRETARY |
| PATCH | `/api/v1/clients/{client_record_id}/correspondence/{correspondence_id}` | Partial update of an entry | ADVISOR, SECRETARY |
| DELETE | `/api/v1/clients/{client_record_id}/correspondence/{correspondence_id}` | Soft-delete an entry | ADVISOR only |

All paths confirmed in `backend/openapi.json`.

**List query parameters:** `page` (default 1), `page_size` (default 20, max 100), `business_id`, `correspondence_type`, `contact_id`, `from_date`, `to_date`, `sort_dir` (`asc`|`desc`, default `desc`).

Router prefix `/clients` is declared in `backend/app/correspondence/api/correspondence.py:23` and mounted under `/api/v1` by `backend/app/router_registry.py`.

## Model & fields

Table: `correspondence_entries`. Source: `backend/app/correspondence/models/correspondence.py`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | No | autoincrement |
| `client_record_id` | int FK → `client_records.id` | No | primary anchor; indexed |
| `business_id` | int FK → `businesses.id` | Yes | optional — UI grouping/filtering only |
| `contact_id` | int FK → `authority_contacts.id` | Yes | optional authority contact link |
| `correspondence_type` | `CorrespondenceType` enum | No | stored as PostgreSQL native enum |
| `subject` | String | No | |
| `notes` | Text | Yes | |
| `occurred_at` | DateTime (timezone=True) | No | stored UTC; frontend renders as Asia/Jerusalem |
| `created_by` | int FK → `users.id` | No | |
| `created_at` | DateTime | No | auto-set via `utcnow_aware` |
| `deleted_at` | DateTime | Yes | soft-delete timestamp |
| `deleted_by` | int FK → `users.id` | Yes | soft-delete actor |

Indexes: `idx_correspondence_business_occurred` (`business_id`, `occurred_at`), `idx_correspondence_occurred` (`occurred_at`), and implicit index on `client_record_id`.

No `updated_at` column — design decision: entries are immutable once created; corrections are soft-deleted and re-entered (see Decisions).

## Enums / statuses

`CorrespondenceType` (`backend/app/correspondence/models/correspondence.py:38`):

| Value | Meaning |
|-------|---------|
| `call` | Phone call (שיחת טלפון) |
| `letter` | Physical letter / registered mail (מכתב / דואר רשום) |
| `email` | Email (דואר אלקטרוני) |
| `meeting` | In-person or video meeting (פגישה פיזית או זום) |
| `fax` | Fax — legally accepted channel with Israeli tax authorities (פקס) |

## Domain rules & invariants

Source: `backend/app/correspondence/services/correspondence_service.py`.

1. **Client record must exist.** Every write and read operation calls `_get_client_record_or_raise` first (`service.py:37-41`). Raises `CLIENT.NOT_FOUND`.
2. **Entry ownership enforced on every access.** `_get_entry_or_raise` checks `entry.client_record_id == client_record_id` (`service.py:55-63`). Cross-client access returns `CORRESPONDENCE.NOT_FOUND`.
3. **Business scope validated on create and update.** When `business_id` is provided, `_assert_business_belongs_to_client` verifies the business belongs to the same legal entity as the client record (`service.py:30-35`). Raises `BUSINESS.NOT_FOUND` on mismatch.
4. **Contact ownership validated on create and update.** When `contact_id` is provided, `_assert_contact_belongs_to_client` checks `contact.client_record_id == client_record_id` (`service.py:43-53`). Raises `CORRESPONDENCE.FORBIDDEN_CONTACT` on mismatch.
5. **`occurred_at` cannot be in the future.** Schema-level `field_validator` rejects future timestamps at request time (`schemas/correspondence.py:10-16`).
6. **Soft delete only.** Delete sets `deleted_at` + `deleted_by`; `get_by_id` and all list queries filter `deleted_at IS NULL` (`repository.py:144-149`, `repository.py:54`).
7. **Delete restricted to ADVISOR.** The DELETE endpoint uses `_ADVISOR_ONLY` dependency (`api/correspondence.py:120`); list/get/create/update allow ADVISOR or SECRETARY.
8. **Default list order descending by `occurred_at`** (latest first); can be reversed with `sort_dir=asc`.

## Error codes

| Code | Raised by | Condition |
|------|-----------|-----------|
| `CORRESPONDENCE.NOT_FOUND` | service `_get_entry_or_raise` | Entry not found or belongs to a different client |
| `CORRESPONDENCE.FORBIDDEN_CONTACT` | service `_assert_contact_belongs_to_client` | `contact_id` does not belong to the requested client record |
| `CLIENT.NOT_FOUND` | service `_get_client_record_or_raise` | `client_record_id` does not exist |
| `BUSINESS.NOT_FOUND` | service / `business_guards.assert_business_belongs_to_legal_entity` | `business_id` not found or belongs to a different legal entity |

Error envelope format: `docs/architecture/error-codes.md`.

## Known issues

**F-CORR-001 (Low) — README documents non-existent error code `CORRESPONDENCE.FORBIDDEN_BUSINESS`.**

`backend/app/correspondence/README.md:129` lists `CORRESPONDENCE.FORBIDDEN_BUSINESS` as a domain error code. The service never raises this code. Business mismatch is surfaced via `BUSINESS.NOT_FOUND` (raised by `business_guards.assert_business_belongs_to_legal_entity:40-43`). The README error code list is incorrect. Fix: remove `CORRESPONDENCE.FORBIDDEN_BUSINESS` from the README.

No IDOR issues found: `update_entry` and `delete_entry` both call `_get_entry_or_raise(entry_id, client_record_id)` before mutating, matching the create-path ownership checks. Business and contact re-validation on update also matches create. All error codes follow `DOMAIN.REASON` format. No stale imports detected.

## Decisions (preserved)

From `backend/app/correspondence/models/correspondence.py` docstring and README:

1. **`client_record_id` is the primary anchor.** Correspondence is tied to the legal entity record, not to a business. `business_id` is optional UI context for grouping.
2. **No `updated_at` column.** Correspondence entries are treated as immutable log records once created. Corrections are made by soft-deleting the original entry and re-entering correct data. This preserves audit integrity.
3. **`occurred_at` stored as UTC.** Frontend is responsible for converting to `Asia/Jerusalem` for display. This avoids DST ambiguity in stored data.
4. **`contact_id` validated against `client_record_id`, not `business_id`.** Authority contacts belong to clients, not businesses. The ownership check uses the client record anchor to avoid incorrect cross-business mismatches.
5. **Soft delete with actor tracking.** `deleted_at` + `deleted_by` support audit requirements for Israeli tax advisory practices.

## Future / planned

None identified. The module is fully implemented. All five endpoints are present in `backend/openapi.json`. Filtering, pagination, and ownership checks are implemented.

## Historical notes

No legacy files under `backend/docs/` existed for this domain. No archive copy created.
