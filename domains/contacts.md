## Scope

This file owns only:

- Canonical current-state documentation for the contacts domain scaffold.

This file must not contain:

- Future contact model, endpoint, or lifecycle behavior that is not implemented.
- Authority-contact behavior.

Source of truth: mandatory

# Contacts

The contacts domain currently exists as a scaffold only. Phase 6 created `backend/app/contacts/` with empty `api/`, `models/`, `repositories/`, `schemas/`, and `services/` package placeholders, plus `backend/tests/contacts/__init__.py`. No contact model, repository, service, schema, router, or API endpoint is implemented yet.

Last verified against code: 2026-06-10.

## Endpoints

No contacts endpoints exist.

## Model & fields

No contacts database model exists.

## Enums / statuses

No contacts enums or statuses exist.

## Domain rules & invariants

- `contacts` is a placeholder package for a future client-people contacts domain.
- Authority contacts are separate and live under `backend/app/authority_contacts/`.
- `Person` records linked to legal entities are not client contact records.

## Error codes

No `CONTACT.*` error codes exist.

## Known issues

- The domain is scaffold-only. Any real contact behavior is future feature work.

## Decisions (preserved)

1. **No extraction was possible in Phase 6.** Code research found no existing client `Contact` model to move.
2. **Authority contacts remain separate.** Do not merge government-authority contacts into this future client-contact domain.

## Future / planned

- Define a real client-contact model, schemas, repository, service, and endpoints only when product requirements are approved.
