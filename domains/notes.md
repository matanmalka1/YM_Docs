## Scope
This file owns only:
- Canonical current-state documentation for the notes domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Notes

The notes domain stores free-text notes attached to backend entities through a shared `entity_notes` table. In current code, the module exposes two scopes only: client notes and business notes, both behind advisor/secretary-only HTTP routes.

Last verified against code + backend/openapi.json: 2026-06-05.

## Endpoints

All routes require role `ADVISOR` or `SECRETARY` via router-level dependencies in `backend/app/notes/api/entity_notes.py:13-17` and `backend/app/notes/api/business_notes.py:13-17`. All paths below exist in `backend/openapi.json:11095-11605`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/clients/{client_id}/notes | List non-deleted notes attached to the client id |
| POST | /api/v1/clients/{client_id}/notes | Create a client note with `created_by` taken from the authenticated user |
| PATCH | /api/v1/clients/{client_id}/notes/{note_id} | Update one client note, but only if the caller created it |
| DELETE | /api/v1/clients/{client_id}/notes/{note_id} | Soft-delete one client note |
| GET | /api/v1/clients/{client_id}/businesses/{business_id}/notes | List non-deleted notes attached to the business after validating the business belongs to the client |
| POST | /api/v1/clients/{client_id}/businesses/{business_id}/notes | Create a business note after validating the business belongs to the client |
| PATCH | /api/v1/clients/{client_id}/businesses/{business_id}/notes/{note_id} | Update one business note after client/business ownership validation |
| DELETE | /api/v1/clients/{client_id}/businesses/{business_id}/notes/{note_id} | Soft-delete one business note after client/business ownership validation |

## Model & fields

Primary model: `EntityNote` in `backend/app/notes/models/entity_note.py:22-35`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | auto-increment (`backend/app/notes/models/entity_note.py:25`) |
| `entity_type` | `String` | no | indexed; identifies the target domain (`backend/app/notes/models/entity_note.py:26`) |
| `entity_id` | int | no | indexed; target row id inside that domain (`backend/app/notes/models/entity_note.py:27`) |
| `note` | `Text` | no | free-text body (`backend/app/notes/models/entity_note.py:28`) |
| `created_by` | int FK -> `users.id` | yes | author id (`backend/app/notes/models/entity_note.py:29`) |
| `created_at` | datetime | no | defaults to `utcnow` (`backend/app/notes/models/entity_note.py:30`) |
| `updated_at` | datetime | yes | set on update via `onupdate=utcnow` (`backend/app/notes/models/entity_note.py:31`) |
| `deleted_at` | datetime | yes | soft-delete marker (`backend/app/notes/models/entity_note.py:32`) |
| `deleted_by` | int FK -> `users.id` | yes | soft-delete actor (`backend/app/notes/models/entity_note.py:33`) |

Indexes: single-column indexes on `entity_type` and `entity_id`, plus composite index `idx_entity_notes_type_id` on `(entity_type, entity_id)` (`backend/app/notes/models/entity_note.py:26-27,35`).

Response-only fields:

- `created_by_name` is attached in the service layer by looking up `created_by` ids in `UserRepository.list_by_ids(...)`; it is not stored on the table (`backend/app/notes/services/entity_note_service.py:17-27`, `backend/app/notes/schemas/entity_note.py:6-16`).

## Enums / statuses

This module defines no SQLAlchemy/Python enum for notes. The real scoped discriminator values used by the HTTP layer are:

| Field | Value | Source |
|------|-------|--------|
| `entity_type` | `client` | `backend/app/notes/api/entity_notes.py:19` |
| `entity_type` | `business` | `backend/app/notes/services/business_note_service.py:14` |

## Domain rules & invariants

- All list queries filter out soft-deleted rows and order notes by `created_at DESC`; pagination defaults to `page=1`, `page_size=50`, with `page_size <= 100` (`backend/app/notes/repositories/entity_note_repository.py:30-51`, `backend/app/notes/api/entity_notes.py:22-41`, `backend/app/notes/api/business_notes.py:20-39`).
- Create/update request bodies contain only one required field, `note: str`; the schema does not define extra validation such as max length or trimming (`backend/app/notes/schemas/entity_note.py:22-27`).
- Updating a note first scopes the lookup by `note_id`, `entity_type`, and `entity_id`; if the note does not belong to that scope, the service raises `NOTE.NOT_FOUND` (`backend/app/notes/services/entity_note_service.py:29-36,69-83`).
- Only the original author may update a note. `update_note()` rejects any caller whose `actor_id` differs from `created_by` with `NOTE.FORBIDDEN` (`backend/app/notes/services/entity_note_service.py:77-80`).
- Delete is a soft delete only. The service confirms that the note belongs to the requested scope, then sets `deleted_at` and `deleted_by`; there is no restore endpoint in this domain (`backend/app/notes/services/entity_note_service.py:85-93`, `backend/app/notes/repositories/entity_note_repository.py:67-74`).
- Business-note endpoints enforce parent ownership before every operation: the business must exist, the client record must exist, and `business.legal_entity_id` must equal the client's `legal_entity_id` (`backend/app/notes/services/business_note_service.py:23-37,39-80,82-89`, `backend/app/businesses/services/business_guards.py:37-43`).

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `NOTE.NOT_FOUND` | Requested note id does not exist, is soft-deleted, or does not belong to the requested `entity_type` + `entity_id` scope (`backend/app/notes/services/entity_note_service.py:29-36,81-82`) |
| `NOTE.FORBIDDEN` | Caller tries to update a note created by another user (`backend/app/notes/services/entity_note_service.py:77-80`) |
| `CLIENT.NOT_FOUND` | Business-note routes: client record does not exist (`backend/app/notes/services/business_note_service.py:88`) |
| `BUSINESS.NOT_FOUND` | Business-note routes: business does not exist or does not belong to the client (`backend/app/notes/services/business_note_service.py:82-89`, `backend/app/businesses/services/business_guards.py:37-43`) |

## Known issues

No open known issues.

## Resolved issues

- **Notes-001** (2026-06-05): Client-note routes did not validate client existence. Fixed: `EntityNoteService._assert_client_exists()` now called on all `list_notes` and `add_note` operations for `entity_type="client"`. Source: `backend/app/notes/services/entity_note_service.py:21-23,53,65`.
- **Notes-002** (2026-06-05): Business-note routes raised `BUSINESS.NOT_FOUND` for a missing client record. Fixed: `business_note_service.py:88` now raises `CLIENT.NOT_FOUND` when the client record does not exist.

## Decisions (preserved)

No notes-specific current decision could be preserved from `backend/docs` or `backend/docs/domain_decisions_v3.md` without inventing behavior. This canonical doc therefore reflects code and OpenAPI only.

## Future / planned

No notes-specific future behavior was found in `backend/docs`. Do not treat restore support, extra entity scopes, or richer note validation as current behavior.

## Historical notes

No `backend/docs` file exists for the notes domain, so this task did not create `docs/archive/notes-legacy.md` and did not replace any legacy notes doc with a pointer.
