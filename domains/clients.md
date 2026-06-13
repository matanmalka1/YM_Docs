## Scope
This file owns only:
- Canonical current-state documentation for the clients domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Clients

The clients domain manages the office's CRM representation of a reporting entity. It owns `ClientRecord`, the operational CRM anchor, and still exposes the routed creation/update workflow that writes the related legal-entity identity graph. The stable tax/legal identity models (`LegalEntity`, `Person`, `PersonLegalEntityLink`) live in the legal-entities domain. All workflow objects (VAT work items, advance payments, annual reports, binders) anchor on `ClientRecord.id`, not on `LegalEntity.id` directly.

Last verified against code + backend/openapi.json: 2026-06-10.

## Endpoints

All endpoints require role `ADVISOR` or `SECRETARY` unless noted. `client_id` in all route params refers to `ClientRecord.id`.

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| POST | /api/v1/clients/preview-impact | Dry-run: returns projected obligations without writing | ADVISOR |
| POST | /api/v1/clients | Create LegalEntity + ClientRecord + first Business in one request | ADVISOR |
| GET | /api/v1/clients | List clients (thin `ClientRecordListItem`; search, status, entity_type, accountant_id, sort, page) | ADVISOR, SECRETARY |
| GET | /api/v1/clients/sidebar | Lightweight list for sidebar nav (page_size ≤ 100); does not include cross-domain alert counts | ADVISOR, SECRETARY |
| GET | /api/v1/clients/{client_record_id} | Get full client by ClientRecord.id | ADVISOR, SECRETARY |
| PATCH | /api/v1/clients/{client_record_id} | Partial update of identity/status fields | ADVISOR, SECRETARY |
| DELETE | /api/v1/clients/{client_record_id} | Soft-delete (ADVISOR only) | ADVISOR |
| POST | /api/v1/clients/{client_record_id}/restore | Restore soft-deleted client | ADVISOR |
| GET | /api/v1/clients/conflict/{id_number} | Active/deleted conflict info for an id_number | ADVISOR, SECRETARY |
| GET | /api/v1/clients/export | Export all clients as Excel workbook | ADVISOR, SECRETARY |
| GET | /api/v1/clients/template | Download blank Excel import template | ADVISOR, SECRETARY |
| POST | /api/v1/clients/import | Bulk-create clients from Excel (max 10 MB, idempotency key required) | ADVISOR |

Cross-domain endpoints scoped to a client (served by other modules):

| Method | Path | Served by |
|--------|------|-----------|
| GET | /api/v1/clients/{client_record_id}/binders | binders domain |
| GET | /api/v1/clients/{client_record_id}/status-card | businesses domain |
| GET | /api/v1/clients/{client_record_id}/annual-reports | annual_reports domain |
| GET/POST/PATCH/DELETE | /api/v1/clients/{client_record_id}/correspondence | communications domain |
| GET/POST/PATCH/DELETE | /api/v1/clients/{client_record_id}/advance-payments | advance_payments domain |
| GET/POST/PATCH/DELETE | /api/v1/clients/{client_record_id}/authority-contacts | authority_contacts domain |
| GET/POST/PATCH/DELETE | /api/v1/clients/{client_record_id}/notes | notes domain |
| GET | /api/v1/clients/{client_record_id}/timeline | timeline domain |
| POST/{client_record_id}/businesses | businesses domain |

(All paths confirmed in backend/openapi.json.)

## Model & fields

### LegalEntity (`backend/app/legal_entities/models/legal_entity.py`)

The stable tax/legal identity. Globally unique by `(id_number_type, id_number)`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| id_number | str | no | ת"ז / ח"פ / passport number |
| id_number_type | IdNumberType | no | see Enums |
| official_name | str | no | |
| entity_type | EntityType | yes | see Enums |
| vat_reporting_frequency | VatType | yes | see Enums |
| advance_payment_frequency | AdvancePaymentFrequency | yes | see Enums |
| vat_exempt_ceiling | Numeric(12,0) | yes | system-set for OSEK_PATUR; cannot be set manually |
| advance_rate | Numeric(5,2) | yes | default rate for obligation generation |
| advance_rate_updated_at | date | yes | |
| annual_revenue | Numeric(15,0) | yes | |
| created_at | datetime | no | |
| updated_at | datetime | yes | |

Constraints: `UniqueConstraint("id_number_type", "id_number")` — global, not soft-delete-aware. Index on `official_name`.

### ClientRecord (`backend/app/clients/models/client_record.py`)

Office CRM anchor. One active record per LegalEntity (enforced by partial unique index on `deleted_at IS NULL`).

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | this is `client_record_id` / `client_id` across the codebase |
| legal_entity_id | int FK → legal_entities.id | no | |
| office_client_number | int | no | sequence from 100001; unique among active records |
| accountant_id | int FK → users.id | yes | |
| status | ClientStatus | no | default `active` |
| notes | text | yes | |
| created_by | int FK → users.id | yes | |
| created_at | datetime | no | |
| updated_at | datetime | yes | |
| deleted_at | datetime | yes | soft-delete column (from SoftDeletableMixin) |

Indexes: partial unique on `(legal_entity_id)` where `deleted_at IS NULL`; partial unique on `(office_client_number)` where `deleted_at IS NULL`; partial index on `created_at DESC` where `deleted_at IS NULL`.

### Person (`backend/app/legal_entities/models/person.py`)

Natural-person identity optionally linked to a LegalEntity. Used for sole traders and similar.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| full_name | str | no | |
| first_name | str | yes | |
| last_name | str | yes | |
| id_number | str | no | |
| id_number_type | IdNumberType | no | constraint: must be `individual`, `passport`, or `other` — not `corporation` |
| phone | str | yes | |
| email | str | yes | |
| address_street | str | yes | |
| address_building_number | str | yes | |
| address_apartment | str | yes | |
| address_city | str | yes | |
| address_zip_code | str | yes | |
| created_at | datetime | no | |
| updated_at | datetime | yes | |

Constraints: `UniqueConstraint("id_number_type", "id_number")`; check constraint prevents `id_number_type = 'corporation'`.

### PersonLegalEntityLink (`backend/app/legal_entities/models/person_legal_entity_link.py`)

Join table associating a Person to a LegalEntity with a role.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | no | |
| person_id | int FK → persons.id | no | |
| legal_entity_id | int FK → legal_entities.id | no | |
| role | PersonLegalEntityRole | no | default `owner` |
| created_at | datetime | no | |

Constraint: `UniqueConstraint("person_id", "legal_entity_id", "role")`.

## Enums / statuses

### ClientStatus (`backend/app/clients/enums.py:4`)

| Value | Meaning |
|-------|---------|
| `active` | Normal CRM state |
| `frozen` | Suspended; open VAT/annual/binder work items are cancelled on transition |
| `closed` | Closed; same cancellation effect as frozen |

### EntityType (`backend/app/common/enums.py`)

| Value | Meaning |
|-------|---------|
| `osek_patur` | עוסק פטור — VAT-exempt; VAT frequency forced to `exempt` |
| `osek_murshe` | עוסק מורשה — collects/deducts VAT |
| `company_ltd` | חברה בע"מ — separate legal entity; must use `corporation` id_number_type |
| `employee` | שכיר — cannot be created as a client via API |

Only `osek_patur`, `osek_murshe`, and `company_ltd` are supported for client creation (`backend/app/clients/constants.py:53`).

### IdNumberType (`backend/app/common/enums.py`)

| Value | Meaning |
|-------|---------|
| `individual` | ת"ז — 9-digit Israeli ID |
| `corporation` | ח"פ — company registration number |
| `passport` | Foreign passport |
| `other` | Other identifier |

### PersonLegalEntityRole (`backend/app/legal_entities/models/person_legal_entity_link.py:14`)

| Value |
|-------|
| `owner` |
| `authorized_signatory` |
| `controlling_shareholder` |

### VatType (`backend/app/common/enums.py`)

`monthly` | `bimonthly` | `exempt`

### AdvancePaymentFrequency (`backend/app/common/enums.py`)

`monthly` | `bimonthly`

## Domain rules & invariants

**Identity & creation** (`backend/app/clients/services/create_client_service.py`, using repositories from `backend/app/legal_entities/repositories/`):

- `EntityType.EMPLOYEE` cannot be created as a client — raises `ValueError` (`create_client_service.py:88`).
- `EntityType.COMPANY_LTD` requires `id_number_type = corporation` — raises `ValueError` if mismatched (`create_client_service.py:154`).
- Creating a client with an `id_number` that already belongs to an active client raises `CLIENT.CONFLICT` (`create_client_service.py:162`).
- Creating a client with an `id_number` of a soft-deleted client raises `CLIENT.DELETED_EXISTS` (`create_client_service.py:166`).
- `OSEK_PATUR` entity type forces `vat_reporting_frequency = exempt` and sets `vat_exempt_ceiling` from the current year's regulatory value; these cannot be supplied or overridden manually (`create_policy.py:17-30`).
- `vat_exempt_ceiling` is system-set for OSEK_PATUR only; the API rejects manual input for any entity type (`constants.py:63-66`).
- `id_number_type` is derived from `entity_type` automatically on creation via `derive_id_number_type` (`create_policy.py:9`).

**One active record per legal entity**: enforced by the partial unique index on `(legal_entity_id) WHERE deleted_at IS NULL` (`client_record.py:51`).

**Onboarding side-effects** (`backend/app/clients/services/client_onboarding_orchestrator.py`): On client creation, the orchestrator automatically:
1. Creates an initial binder.
2. Generates tax obligations.
3. Creates pending VAT work items for future periods (based on `vat_reporting_frequency`).
4. Creates advance payment records for future periods (based on `advance_payment_frequency`).

**Status transitions** (`backend/app/clients/services/client_update_service.py:76`): When status transitions to `frozen` or `closed`, open VAT work items, open annual reports, and in-office binders are cancelled via their respective domain services.

**Entity type changes** (`backend/app/clients/services/client_update_service.py:43`): Only `ADVISOR` role may change `entity_type`. The change is audit-logged with `ACTION_ENTITY_TYPE_CHANGED`. Obligation regeneration is triggered on any change to `entity_type`, `vat_reporting_frequency`, or `advance_payment_frequency` (`constants.py:46`).

**Guard** (`backend/app/clients/guards/client_record_guards.py`): `assert_client_record_is_active` raises `CLIENT_RECORD.CLOSED` if status is `frozen` or `closed`. Used by downstream domains before creating work items.

**Workflow anchor rule** (from `domain_decisions_v3.md`, decision #4): All workflow objects (AdvancePayment, VatWorkItem, AnnualReport, Binder) anchor on `client_record_id`, never directly on `legal_entity_id`.

**Domain independence rule** (from `domain_decisions_v3.md`, decision #2): VAT frequency and advance payment frequency are independent. Never derive one from the other.

**Soft delete**: `ClientRecord` uses `SoftDeletableMixin` (adds `deleted_at`, `deleted_by`, `restored_at`, `restored_by`). Hard deletes are not supported. Restore checks that no active client already holds the same `id_number` before restoring.

**Excel import**: partial-success semantics — valid rows are created, invalid rows are returned in an `errors` array. Requires `X-Idempotency-Key` header. Max upload size: 10 MB. Default entity type for import rows: `osek_murshe`; default VAT frequency: `bimonthly`; default advance payment frequency: `bimonthly` (`constants.py:41-43`).

**List payload (thin DTO)**: `GET /api/v1/clients` returns the thin `ClientRecordListItem` (identity, status, contact, `office_client_number`, `active_binder_number`, `created_at`) — only the fields the clients table renders. Full profile fields (tax-reporting config, address, advance-rate, `annual_revenue`, `annual_turnover`, metadata) are served only by `GET /api/v1/clients/{client_record_id}` as `ClientRecordResponse`; the list-page edit form fetches the client by id before editing. Because turnover is detail-only, the list no longer performs the per-page annual-turnover lookup and the list endpoint no longer accepts the (previously turnover-only) `tax_year` query param (`backend/app/clients/services/client_query_service.py`, `backend/app/clients/services/client_enrichment_service.py`).

**Sidebar payload**: `GET /api/v1/clients/sidebar` returns only client identity/navigation fields and pagination metadata. The service delegates to `ClientRecordRepository.list_sidebar()` and `count_sidebar()`, so it currently does not attach notification, work-queue, or other cross-domain alert counts (`backend/app/clients/api/clients.py:115-131`, `backend/app/clients/services/client_query_service.py:132-153`, `backend/app/clients/repositories/client_record_repository.py:222-262`).

## Error codes

Registry: `docs/backend/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `CLIENT.NOT_FOUND` | `client_id` does not exist or is already deleted (`client_lifecycle_service.py:27`) |
| `CLIENT.CONFLICT` | `id_number` already belongs to an active client (`create_client_service.py:163`) |
| `CLIENT.DELETED_EXISTS` | `id_number` belongs to a soft-deleted client (`create_client_service.py:168`) |
| `CLIENT.NOT_DELETED` | Restore called on a client that is not deleted (`client_lifecycle_service.py:36`) |
| `CLIENT.ENTITY_TYPE_CHANGE_FORBIDDEN` | Non-ADVISOR attempts to change `entity_type` (`client_update_service.py:48`) |
| `CLIENT_RECORD.CLOSED` | Downstream action attempted on a frozen/closed client (`client_record_guards.py:6`) |

## Known issues

None found during authoring.

## Decisions (preserved)

From `backend/docs/domain_decisions_v3.md` (still true, verified against code):

1. **Workflow anchor = client_record_id** (decision #4): All workflow objects anchor on `ClientRecord.id`, not on `LegalEntity.id`. Joins to LegalEntity always go through ClientRecord. This is the most important architectural invariant — breaking it causes subtle join bugs.

2. **Domain independence** (decision #2): `vat_reporting_frequency` and `advance_payment_frequency` are completely independent. No derivation between them is allowed.

3. **Soft delete is a domain contract**: Avoid hard deletes unless there is a migration/maintenance requirement. The partial unique indexes are soft-delete-aware.

4. **LegalEntity uniqueness is global** (not soft-delete-aware): `UniqueConstraint("id_number_type", "id_number")` on `legal_entities` has no `WHERE deleted_at IS NULL` clause. A legal entity with a given id_number can only ever exist once in the table, even if its ClientRecord is deleted.

5. **office_client_number sequence starts at 100001**: `client_records` table uses a PostgreSQL sequence. In test environments, the sequence is simulated via `before_insert` event to avoid sequence exhaustion (`client_record.py:78`).

## Future / planned

- Batched sidebar alert counts are not implemented. The current frontend client sidebar loads up to 100 clients from `/api/v1/clients/sidebar` and leaves an explicit TODO for alerts until a navigation-specific count surface exists (`frontend/src/components/layout/ClientSidebar/useClientSidebarClients.ts:7-14`, `frontend/src/components/layout/ClientSidebar/ClientSidebar.tsx:129-132`).
- Removal of `AdvancePayment.due_date` (the legacy column): per `domain_decisions_v3.md` decision #10, migration to `due_date_effective` as the single source of truth is in progress; the old column is still present in code.
- `NATIONAL_INSURANCE` obligation type in `common/enums.py` is reserved but not yet wired to a `DeadlineRuleType` — intentionally unsupported.

## Historical notes

No legacy files in `backend/docs` covered the clients domain specifically. The `backend/app/clients/README.md` file served as the pre-existing domain reference. It has been superseded by this document; the README has been updated to pointer-only.
