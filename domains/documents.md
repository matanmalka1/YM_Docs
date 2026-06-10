## Scope

This file owns only:

- Canonical current-state documentation for the documents parent package.

This file must not contain:

- Permanent-document behavior details owned by `docs/domains/permanent-documents.md`.
- Future generic document behavior that is not implemented.

Source of truth: mandatory

# Documents

The documents domain is currently a parent package. Phase 5 moved the implemented permanent-documents domain under `backend/app/documents/permanent_documents/`. There is no generic `Document` model, router, service, schema, or repository at the parent `backend/app/documents/` level.

Last verified against code: 2026-06-10.

## Endpoints

No generic documents endpoints exist. Permanent-document endpoints are still owned by the permanent-documents child domain and documented in `docs/domains/permanent-documents.md`.

## Model & fields

No generic documents database model exists.

## Enums / statuses

No generic documents enums or statuses exist.

## Domain rules & invariants

- `backend/app/documents/` is a parent namespace.
- The implemented child domain is `backend/app/documents/permanent_documents/`.
- Permanent-document API routes and table names did not change in Phase 5.

## Error codes

No generic `DOCUMENT.*` error codes exist.

## Known issues

- The parent domain is structural only. Generic document behavior is not implemented.

## Decisions (preserved)

1. **Parent package only.** Phase 5 created the documents parent by relocating permanent documents; it did not create a generic document aggregate.
2. **Child domain remains authoritative.** Permanent-document behavior remains documented in `docs/domains/permanent-documents.md`.

## Future / planned

- Add parent-level document behavior only if a real generic document domain is designed and implemented.
