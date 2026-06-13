## Scope
This file owns only:
- Canonical current-state documentation for the audit domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Audit

The audit domain stores append-only change records for selected business entities and exposes one read-only HTTP endpoint for staff to inspect those records. In the current implementation, writes happen indirectly through `EntityAuditWriter` from other domains; the `audit` module itself owns the generic table, response schema, repository query path, and entity-type validation for read access.
Last verified against code + backend/openapi.json: 2026-06-11.

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
| `performed_by` | `int` FK -> `users.id` | no | actor who performed the change |
| `action` | `String` | no | free string action name |
| `old_value` | `Text` | yes | JSON string snapshot before mutation |
| `new_value` | `Text` | yes | JSON string snapshot after mutation |
| `note` | `Text` | yes | optional audit note |
| `performed_at` | `datetime` | no | defaults to `utcnow` |

There is also a composite index `idx_entity_audit_type_id` on (`entity_type`, `entity_id`). Source: `backend/app/audit/models/entity_audit_log.py:24-44`.

### Response schema

`EntityAuditLogResponse` exposes the stored fields plus `performed_by_name`; `EntityAuditTrailResponse` wraps `items`, `total`, `page`, and `page_size`. Source: `backend/app/audit/schemas/entity_audit_log.py:8-27`; OpenAPI components: `backend/openapi.json`.

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
- `performed_by_name` is not stored on the audit row; it is enriched at read time by bulk-loading distinct user ids from `users` and mapping them onto the response items. Source: `backend/app/audit/services/audit_trail_service.py:63-74`.
- Audit rows are append-only by design: the model has no soft-delete columns and no update path in this module. Corrections are represented by new rows, not edits to existing ones. Source: `backend/app/audit/models/entity_audit_log.py:4-12,24-44`.
- `EntityAuditWriter.append()` is a no-op when `actor_id` is `None`; callers only get an audit row when a concrete actor id is available. Source: `backend/app/audit/services/entity_audit_writer.py:26-47`.
- `EntityAuditWriter` serializes payload snapshots to JSON strings, normalizing enums to `.value`, datetimes/dates to ISO strings, decimals to strings, and wrapping raw string payloads as `{"value": ...}` before persistence. Source: `backend/app/audit/services/entity_audit_writer.py:138-159`.

## Error codes

The audit module raises these `AUDIT.*` codes:

- `AUDIT.INVALID_ENTITY_TYPE` for unsupported read targets. Source: `backend/app/audit/services/audit_trail_service.py:35-37`.
- `AUDIT.ENTITY_NOT_FOUND` when the requested entity id does not exist for the validated type. Source: `backend/app/audit/services/audit_trail_service.py:50-53`.

Registry: `docs/architecture/error-codes.md`.

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
