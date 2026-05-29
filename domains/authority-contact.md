## Scope
This file owns only:
- Canonical current-state documentation for the authority-contact domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Authority Contact

The authority-contact domain manages named contacts at Israeli government authorities (Tax Authority / פקיד שומה, VAT branches / סניף מע"מ, National Insurance / ביטוח לאומי) that a tax advisor maintains for each client. Contacts are scoped to a `client_record` and may be referenced by the `correspondence` domain when logging interactions with those authorities.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

| Method | Path | Purpose | Roles |
|--------|------|---------|-------|
| POST | `/api/v1/clients/{client_record_id}/authority-contacts` | Create a new contact for a client | ADVISOR, SECRETARY |
| GET | `/api/v1/clients/{client_record_id}/authority-contacts` | List contacts for a client (paginated, filterable by type) | ADVISOR, SECRETARY |
| GET | `/api/v1/clients/authority-contacts/{contact_id}` | Fetch a single contact by ID | ADVISOR, SECRETARY |
| PATCH | `/api/v1/clients/authority-contacts/{contact_id}` | Partially update a contact | ADVISOR, SECRETARY |
| DELETE | `/api/v1/clients/authority-contacts/{contact_id}` | Soft-delete a contact | ADVISOR only |

All paths confirmed in `backend/openapi.json`. Router prefix `/clients` is set in `backend/app/authority_contact/api/authority_contact.py:16`; mounted under `/api/v1` via `router_registry`.

## Model & fields

**Table:** `authority_contacts` — `backend/app/authority_contact/models/authority_contact.py`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | autoincrement |
| `client_record_id` | int FK → `client_records.id` | no | indexed |
| `contact_type` | `ContactType` enum | no | indexed (`idx_authority_contact_type`) |
| `name` | String | no | |
| `office` | String | yes | branch / department name |
| `phone` | String | yes | |
| `email` | String | yes | validated as `EmailStr` on input |
| `notes` | Text | yes | |
| `created_at` | datetime | no | set on insert via `utcnow` |
| `updated_at` | datetime | yes | set on update via `onupdate=utcnow` |
| `deleted_at` | datetime | yes | soft-delete timestamp |
| `deleted_by` | int FK → `users.id` | yes | audit: who deleted |

Source: `backend/app/authority_contact/models/authority_contact.py:48–72`.

## Enums / statuses

**`ContactType`** — `backend/app/authority_contact/models/authority_contact.py:41–45`

| Value | Hebrew label |
|-------|-------------|
| `assessing_officer` | פקיד שומה |
| `vat_branch` | סניף מע"מ |
| `national_insurance` | ביטוח לאומי |
| `other` | אחר |

## Domain rules & invariants

- **Client-record existence check on create/list.** `add_contact` and `list_client_contacts` validate the `client_record_id` via `ClientRecordRepository.get_by_id`; raise `CLIENT.NOT_FOUND` if missing. Source: `backend/app/authority_contact/services/authority_contact_service.py:32–33, 69–70`.
- **Soft delete, never hard delete.** `delete_contact` sets `deleted_at` + `deleted_by`; the row is preserved for audit. Source: `backend/app/authority_contact/repositories/authority_contact_repository.py:81–96`.
- **Deleted rows excluded from all reads.** `_base_where` always includes `deleted_at IS NULL`; `BaseRepository.get_by_id` also applies this filter. Source: `backend/app/authority_contact/repositories/authority_contact_repository.py:41–47`, `backend/app/common/repositories/base_repository.py:17–20`.
- **List ordered newest-first.** `list_by_client_record` orders by `created_at DESC`. Source: `backend/app/authority_contact/repositories/authority_contact_repository.py:57–63`.
- **Delete role-restricted to ADVISOR.** The DELETE endpoint overrides the router-level role with `require_role(UserRole.ADVISOR)`. Source: `backend/app/authority_contact/api/authority_contact.py:115`.
- **Partial updates.** `AuthorityContactUpdateRequest` uses `model_dump(exclude_unset=True)`, so only provided fields are written. Source: `backend/app/authority_contact/api/authority_contact.py:107`.

## Error codes

| Code | Trigger |
|------|---------|
| `CLIENT.NOT_FOUND` | `client_record_id` does not exist (create, list) |
| `AUTHORITY_CONTACT.NOT_FOUND` | `contact_id` not found or already soft-deleted (get, update, delete) |

Registry: `docs/architecture/error-codes.md`.

## Known issues

### F-001 — IDOR on GET, PATCH, DELETE contact endpoints

**What is wrong:** `get_contact`, `update_contact`, and `delete_contact` resolve the contact by bare `contact_id` with no ownership check against any `client_record_id`. Any authenticated ADVISOR or SECRETARY can read or update any contact system-wide by guessing/enumerating IDs; any ADVISOR can delete any contact.

**Path:line:**
- `backend/app/authority_contact/services/authority_contact_service.py:45–58` (`update_contact` — no ownership check)
- `backend/app/authority_contact/services/authority_contact_service.py:84–88` (`get_contact` — no ownership check)
- `backend/app/authority_contact/services/authority_contact_service.py:78–82` (`delete_contact` — no ownership check)
- `backend/app/authority_contact/repositories/authority_contact_repository.py:78` (`update` calls `get_by_id` — only filters by `id` + `deleted_at`)
- `backend/app/common/repositories/base_repository.py:48–49` (`get_by_id` is ID-only)

**Rule violated:** A contact belongs to `client_record_id`. Operations on a contact by ID must verify the contact's `client_record_id` matches the calling context (or that the user has access to that client record).

**Suggested fix:** Pass `client_record_id` from the API layer into `get_contact`, `update_contact`, and `delete_contact`; add a `WHERE contact_id = :id AND client_record_id = :client_record_id` check, or fetch and assert `contact.client_record_id == expected_client_record_id`.

This matches the recurring IDOR pattern documented in `docs/project/security-findings.md` (annual-reports F-001, vat-reports F-008).

### F-002 — `AuthorityContactLink` described in model docstring but not implemented

**What is wrong:** The model file docstring (`backend/app/authority_contact/models/authority_contact.py:12–27`) documents an `AuthorityContactLink` many-to-many table with a `UniqueConstraint on (contact_id, client_id, business_id)` and an optional `business_id` column. Neither the `AuthorityContactLink` model nor the `business_id` column exists anywhere in the codebase.

**Path:line:** `backend/app/authority_contact/models/authority_contact.py:12–27`.

**Rule violated:** Docs must not describe unimplemented models as if they exist (pattern 5 — broken/stale references).

**Suggested fix:** Either implement `AuthorityContactLink` and `business_id`, or remove the stale docstring. Until implemented, see Future / planned below.

## Decisions (preserved)

From `backend/app/authority_contact/README.md` (last audited 2026-04-13):

- Contacts are `client_record`-scoped (not business-scoped) in the current implementation.
- `correspondence` references a contact via `contact_id`; the validation checks that the contact's `client_record_id` matches the correspondence `client_record_id` (not business ownership).
- No soft delete on link records — links are simply deleted when removed (design intent; `AuthorityContactLink` not yet implemented).
- `updated_at` tracked on `AuthorityContact` because contact details (phone, office) change over time.

## Future / planned

- **`AuthorityContactLink` many-to-many sharing.** The model docstring describes a future `AuthorityContactLink` table allowing a single contact record to be shared across multiple clients/businesses, with an optional `business_id` column for VAT-branch-specific contacts. A `UniqueConstraint(contact_id, client_id, business_id)` is described. **Not implemented.** Source: `backend/app/authority_contact/models/authority_contact.py:12–27`.
- **`business_id` on `AuthorityContact`.** The same docstring notes that `business_id` will be the primary association (the business that "owns" the contact), with sharing handled by `AuthorityContactLink`. **Not implemented.**

## Historical notes

No legacy `backend/docs` files exist for this domain. No archive copy was created.
