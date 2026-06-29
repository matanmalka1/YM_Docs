## Scope
This file owns only:
- Canonical current-state documentation for the audit domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Audit

The audit domain stores append-only change records for selected business entities and exposes one read-only HTTP endpoint for staff to inspect those records. In the current implementation, writes happen indirectly through `EntityAuditWriter` from other domains; the `audit` module itself owns the generic table, response schema, repository query path, and entity-type validation for read access.
Last verified against code + backend/openapi.json: 2026-06-29.

## Two audit models (target shape)

The audit refactor (see `docs/audit-refactor-implementation-plan.md`) converges on exactly two audit models with **distinct shapes** — do not conflate them:

- **`EntityAuditLog`** (table `entity_audit_logs`, this domain) — business mutations + evidence events. Carries `actor_type`, `actor_display_name`, and JSON-object `old_value`/`new_value`/`metadata_json`. `performed_by` is nullable (system / external-signer rows have no `users.id`).
- **`UserAuditLog`** (table `user_audit_logs`, owned by the `users` domain) — auth/security/admin-access events. Has **no** `old_value`/`new_value` and **no** `actor_type`; it stores a JSON-object `metadata_json` plus `actor_display_name` + `target_display_name` snapshots and the closed `AuditAction`/`AuditStatus` enums.

**Actor snapshot convention (§5 of the plan).** Display names are immutable snapshots captured at **write time** from the route's `current_user.full_name` and threaded alongside the actor id — never joined from `users` at read time (a rename must not rewrite historical audit display). `EntityAuditLog.actor_type` is one of `user | system | external_signer`; for `system`/`external_signer` rows `performed_by` is `NULL` and `actor_display_name` carries the system label / signer name. As of Phase 1 every existing `EntityAuditLog` write passes `actor_type` (default `"user"`) + `actor_display_name`, and the `UserAuditLog` auth/admin writers capture `actor_display_name` (+ `target_display_name` where a target user exists).

**JSON-object storage convention.** `old_value`/`new_value`/`metadata_json` are stored as JSON objects (PostgreSQL `JSONB`, SQLite `JSON`) using the portable `JSON().with_variant(JSONB, "postgresql")` column type. The writer persists dict/list values directly — it does **not** `json.dumps` them into strings — and readers/response schemas expose them as JSON objects (`dict | list | null`), not strings.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/v1/audit/{entity_type}/{entity_id}` | Return paginated audit entries for one supported entity instance. |

The router is defined with `prefix="/audit"` at `backend/app/audit/api/routes.py:10-27`, and the effective path exists in `backend/openapi.json:11607-11680`.

## Model & fields

### `EntityAuditLog` — table `entity_audit_logs`

The audit domain owns one persisted SQLAlchemy model:

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `id` | `int` PK | no | autoincrement primary key |
| `entity_type` | `String` | no | free string, indexed |
| `entity_id` | `int` | no | target entity id, indexed |
| `performed_by` | `int` FK -> `users.id` | **yes** | actor id; `NULL` for system / external-signer rows |
| `actor_type` | `String` | no | `user` \| `system` \| `external_signer` (model default `"user"`) |
| `actor_display_name` | `String` | yes | immutable actor-name snapshot captured at write time |
| `action` | `String` | no | free string action name |
| `old_value` | `JSONB` (SQLite `JSON`) | yes | JSON-object snapshot before mutation |
| `new_value` | `JSONB` (SQLite `JSON`) | yes | JSON-object snapshot after mutation |
| `metadata_json` | `JSONB` (SQLite `JSON`) | yes | structured context (e.g. `client_record_id`) |
| `note` | `Text` | yes | optional audit note |
| `performed_at` | `datetime` | no | defaults to `utcnow` |

Indexes: `(entity_type, entity_id, performed_at)`, `(action, performed_at)`, `(performed_by, performed_at)`, `(performed_at)`, plus the PostgreSQL expression index `idx_entity_audit_client_ctx` on `((metadata_json->>'client_record_id'), performed_at)` (created in migration; SQLite dev may table-scan). Source: `backend/app/audit/models/audit_entity_audit_log.py`; migrations `backend/alembic/versions/0002_audit_jsonb_actor.py` + `0003_audit_actor_type_notnull.py`.

### Response schema

`EntityAuditLogResponse` exposes the stored fields (`old_value`/`new_value`/`metadata_json` as JSON objects, `actor_type`, `actor_display_name`, nullable `performed_by`) plus the read-time-enriched `performed_by_name`; `EntityAuditTrailResponse` wraps `items`, `total`, `page`, and `page_size`. Source: `backend/app/audit/schemas/audit_entity_audit_log.py`; OpenAPI components: `backend/openapi.json`.

## Enums / statuses

This domain does not use Python enums for audit entity types or actions. Current exact string constants are:

- Readable `entity_type` values: `annual_report`, `business`, `charge`, `client`. Source: `backend/app/audit/constants.py:8-18`.
- Shared action constants: `created`, `updated`, `deleted`, `restored`, `entity_type_changed`. Source: `backend/app/audit/constants.py:23-28`.
- Charge action constants: `issued`, `paid`, `canceled`. Source: `backend/app/audit/constants.py:30-33`.
- Annual-report action constants: `status_changed`, `annual_report_detail_updated`, `annual_report_deadline_updated`, `annex_line_added`, `annex_line_updated`, `annex_line_deleted`, `income_added`, `income_updated`, `income_deleted`, `expense_added`, `expense_updated`, `expense_deleted`. Source: `backend/app/audit/constants.py:35-49`.

## Domain rules & invariants

- The only HTTP route is read-only and role-gated to `ADVISOR` and `SECRETARY`. Source: `backend/app/audit/api/routes.py:13-27`.
- The read API rejects any `entity_type` not present in `ALLOWED_READ_ENTITY_TYPES` with `AUDIT.INVALID_ENTITY_TYPE`. Source: `backend/app/audit/services/audit_trail_service.py:35-37`.
- Read access first verifies that the target entity exists in its owning repository (`ClientRecord`, `Business`, `Charge`, or `AnnualReport`) and raises `AUDIT.ENTITY_NOT_FOUND` with HTTP 404 when absent. Source: `backend/app/audit/services/audit_trail_service.py:39-53`.
- The read API accepts `page` and `page_size` query parameters; the service translates them to repository `limit`/`offset` internally. It also accepts optional filters `action` (str), `user_id` (int, matched against `performed_by`), and `created_after`/`created_before` (datetime, matched against `performed_at`). Audit trail queries are always scoped to the exact `(entity_type, entity_id)` pair, narrowed by any supplied filters, and ordered newest-first by `performed_at DESC, id DESC`. Source: `backend/app/audit/api/routes.py:18-27`, `backend/app/audit/services/audit_trail_service.py:55-75`, `backend/app/audit/repositories/entity_audit_log_repository.py:37-61`.
- Display reads **prefer the immutable `actor_display_name` snapshot** over the live-join `performed_by_name`. `performed_by_name` is not stored on the audit row; it is enriched at read time by bulk-loading distinct user ids from `users` (so it tracks renames), and is exposed only as a fallback for legacy rows without a snapshot. A user rename therefore changes `performed_by_name` but never `actor_display_name`. Source: `backend/app/audit/services/audit_trail_service.py`; frontend `AuditTrailTable` prefers `actor_display_name → performed_by_name → #id → —`.
- Audit rows are append-only by design: the model has no soft-delete columns and no update path in this module. Corrections are represented by new rows, not edits to existing ones. Source: `backend/app/audit/models/entity_audit_log.py:4-12,24-44`.
- `EntityAuditWriter.append()` is a no-op when `actor_id` is `None`; callers only get an audit row when a concrete actor id is available (system / external-signer actor handling via `actor_type` is a later phase). Source: `backend/app/audit/services/audit_entity_audit_writer_service.py`.
- `EntityAuditWriter` normalizes payload snapshots (enums to `.value`, datetimes/dates to ISO strings, decimals to strings, raw string payloads wrapped as `{"value": ...}`) and stores the resulting **dict/list object directly** in the JSONB column — it no longer `json.dumps` snapshots into strings. Source: `backend/app/audit/services/audit_entity_audit_writer_service.py`.
- Every `EntityAuditWriter` write passes `actor_type` (default `"user"`) and threads `actor_display_name` from the route's `current_user.full_name`. Source: the 30 existing write sites across clients/businesses/charges/annual_reports.

## Error codes

The audit module raises these `AUDIT.*` codes:

- `AUDIT.INVALID_ENTITY_TYPE` for unsupported read targets. Source: `backend/app/audit/services/audit_trail_service.py:35-37`.
- `AUDIT.ENTITY_NOT_FOUND` when the requested entity id does not exist for the validated type. Source: `backend/app/audit/services/audit_trail_service.py:50-53`.

Registry: `docs/backend/error-codes.md`.

## Known issues

No open known issues.

## Decisions (preserved)

Still-true decisions preserved from the shared historical reference `backend/docs/history-vs-timeline.md`:

1. `AuditTrail` is the accountability log for audited entity changes, separate from the operational `Timeline`. Source: `backend/docs/history-vs-timeline.md:18-22`.
2. The system should not dump the full audit stream into timeline views; timeline may use only selected high-value audit-derived events, while audit remains the complete record. Source: `backend/docs/history-vs-timeline.md:20-22`.
3. Generic audit inspection is intentionally scoped around the entity families `client`, `business`, `charge`, and `annual_report`, which matches the current `ALLOWED_READ_ENTITY_TYPES`. Source: `backend/docs/history-vs-timeline.md:24-29`; `backend/app/audit/constants.py:8-18`.

## Future / planned

- A generic entity-audit UI can be reused for `client`, `business`, `charge`, and `annual_report`; this is described in the shared historical note but is not implemented in `backend/app/audit` itself. Source: `backend/docs/history-vs-timeline.md:24-29`.
- The timeline may later include selected audit-derived events, but not the complete audit stream. This is future intent only, not a current integration contract. Source: `backend/docs/history-vs-timeline.md:22`.

## Historical notes

No audit-specific `backend/docs/**` legacy file was found to archive. The only relevant legacy material was the shared conceptual reference `backend/docs/history-vs-timeline.md`, which was left unchanged because it is not audit-exclusive and also informs the timeline boundary.
