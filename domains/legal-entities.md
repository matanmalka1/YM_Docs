## Scope

This file owns only:

- Canonical current-state documentation for the legal-entities domain.

This file must not contain:

- Architecture/API rules (link to docs/architecture/*).
- Client lifecycle, onboarding, or ownership behavior.

Source of truth: mandatory

# Legal Entities

The legal-entities domain owns the stable tax/legal identity graph: `LegalEntity`, `Person`, and `PersonLegalEntityLink`. It is an internal backend domain after Phase 1 extraction; it has models and repositories under `backend/app/legal_entities/`, but no HTTP router or service layer yet. Client creation and update routes still live in the clients domain and orchestrate writes to this identity graph.

Last verified against code + backend/openapi.json: 2026-06-10.

## Endpoints

No dedicated legal-entities endpoints exist. The current API surface for creating and updating legal-entity identity data is still exposed through the clients routes documented in `docs/domains/clients.md`.

## Model & fields

### LegalEntity (`backend/app/legal_entities/models/legal_entity.py`)

The stable legal/tax identity. Globally unique by `(id_number_type, id_number)`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| id_number | str | no | Israeli ID / corporation number / passport / other identifier |
| id_number_type | IdNumberType | no | see Enums |
| entity_type | EntityType | yes | see Enums |
| official_name | str | no | |
| vat_reporting_frequency | VatType | yes | see Enums |
| advance_payment_frequency | AdvancePaymentFrequency | yes | see Enums |
| vat_exempt_ceiling | Numeric(12,0) | yes | system-set for OSEK_PATUR through client creation policy |
| advance_rate | Numeric(5,2) | yes | default rate for advance-payment generation |
| advance_rate_updated_at | date | yes | |
| annual_revenue | Numeric(15,0) | yes | |
| created_at | datetime | no | |
| updated_at | datetime | yes | |

Constraints: `UniqueConstraint("id_number_type", "id_number")`; index on `official_name`.

### Person (`backend/app/legal_entities/models/person.py`)

Physical individual who may own or be linked to one or more legal entities.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| full_name | str | no | |
| first_name | str | yes | |
| last_name | str | yes | |
| id_number | str | no | |
| id_number_type | IdNumberType | no | must be `individual`, `passport`, or `other` |
| phone | str | yes | |
| email | str | yes | |
| address_street | str | yes | |
| address_building_number | str | yes | |
| address_apartment | str | yes | |
| address_city | str | yes | |
| address_zip_code | str | yes | |
| created_at | datetime | no | |
| updated_at | datetime | yes | |

Constraints: `UniqueConstraint("id_number_type", "id_number")`; check constraint rejects `corporation`; indexes on `full_name` and `id_number`.

### PersonLegalEntityLink (`backend/app/legal_entities/models/person_legal_entity_link.py`)

Join table associating a `Person` to a `LegalEntity` with a role.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| person_id | int FK -> persons.id | no | |
| legal_entity_id | int FK -> legal_entities.id | no | |
| role | PersonLegalEntityRole | no | defaults to `owner` |
| created_at | datetime | no | |

Constraint: `UniqueConstraint("person_id", "legal_entity_id", "role")`; index on `legal_entity_id`.

## Enums / statuses

Shared enum definitions live in `backend/app/common/enums.py`.

| Enum | Values |
|------|--------|
| `EntityType` | `osek_patur`, `osek_murshe`, `company_ltd`, `employee` |
| `IdNumberType` | `individual`, `corporation`, `passport`, `other` |
| `VatType` | `monthly`, `bimonthly`, `exempt` |
| `AdvancePaymentFrequency` | `monthly`, `bimonthly` |

`PersonLegalEntityRole` is defined in `backend/app/legal_entities/models/person_legal_entity_link.py`: `owner`, `authorized_signatory`, `controlling_shareholder`.

## Domain rules & invariants

- `LegalEntityRepository` owns legal-entity persistence: create, get by id, list by ids, and get by `(id_number_type, id_number)`.
- `PersonRepository` owns person persistence, owner lookup, and owner-link creation. `ensure_owner()` maps corporation owner identity to `IdNumberType.OTHER` before creating/finding a `Person`.
- The Phase 1 extraction was structural only: table names, constraints, repository method signatures, service behavior, and API contracts did not change.
- `LegalEntity` uniqueness is global, not soft-delete-aware. A legal identity with the same `(id_number_type, id_number)` can exist only once in the `legal_entities` table.
- Workflow objects still anchor on `client_record_id`, never directly on `legal_entity_id`; cross-domain legal-entity reads go through `ClientRecord` unless a repository explicitly needs a legal-entity join for scoping or projection.

## Error codes

There are no dedicated `LEGAL_ENTITY.*` error codes today. Current conflicts and not-found cases are surfaced through the clients domain while clients owns the routed creation/update workflow.

## Known issues

- No dedicated service or HTTP router exists for legal entities. Identity writes are still orchestrated by clients services.

## Decisions (preserved)

1. **Legal entity is first-class identity ownership.** `LegalEntity`, `Person`, `PersonLegalEntityLink`, `LegalEntityRepository`, and `PersonRepository` live under `backend/app/legal_entities/` after Phase 1.
2. **No table rename.** The existing `legal_entities`, `persons`, and `person_legal_entity_links` tables already use target names and were not renamed.
3. **No API contract change.** Clients routes remain the public API for creating/updating identity data until a later phase explicitly introduces a legal-entities API.

## Future / planned

- Add a legal-entities service/router only when the product needs legal-entity management independent of client onboarding.
- Consider moving legal-entity-specific update policy out of clients services in a later phase; do not add compatibility aliases for old import paths.
