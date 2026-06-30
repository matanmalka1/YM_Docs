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

The router is defined with `prefix="/audit"` in `backend/app/audit/api/audit_routes.py`, and the effective path is exported in `backend/openapi.json`.

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

`EntityAuditLogResponse` exposes the stored fields (`old_value`/`new_value`/`metadata_json` as JSON objects, `actor_type`, `actor_display_name`, nullable `performed_by`) plus the read-time-enriched `performed_by_name`; `EntityAuditTrailResponse` wraps `items`, `total`, `page`, `page_size`, and an envelope-level `entity_deleted: bool` (true when the audited entity is soft- or hard-deleted; history stays readable). Source: `backend/app/audit/schemas/audit_entity_audit_log.py`; OpenAPI components: `backend/openapi.json`.

## Enums / statuses

This domain does not use Python enums for audit entity types or actions. As of Phase 2, persisted `action` values are **namespaced** `<entity_type>.<verb>` (never bare). Constants live in `backend/app/audit/audit_constants.py`:

- Entity-type constants `ENTITY_*` cover the full §6 set (23 types) and are the canonical keys for the registry.
- Generic verbs (`created`/`updated`/`deleted`/`restored`/`status_changed`) are *bare* building blocks; the writer composes the namespace via `entity_action(entity_type, verb)` → e.g. `client.created`, `annual_report.status_changed`. `client.entity_type_changed` is a first-class semantic action (`ACTION_ENTITY_TYPE_CHANGED`).
- Charge-specific actions are pre-namespaced: `charge.issued`, `charge.paid`, `charge.canceled` (`ACTION_CHARGE_*`).
- Annual-report actions are pre-namespaced: `annual_report.updated` (detail edits), `annual_report.deadline_updated`, `annual_report.{income,expense}_line_{added,updated,deleted}`, `annual_report.annex_line_{added,updated,deleted}`.

`ALLOWED_READ_ENTITY_TYPES` is **derived from the registry** (`allowed_read_entity_types()`), never hand-maintained.

## Domain rules & invariants

- The only HTTP route is read-only and role-gated to `ADVISOR` and `SECRETARY`. Both roles may read every audit trail; `scope_to_active_clients_stmt` is a live-listing deleted-client filter, NOT authorization, and audit reads bypass it so deleted history stays readable. Source: `backend/app/audit/api/audit_routes.py`.
- The read API rejects any `entity_type` not present in the registry with `AUDIT.INVALID_ENTITY_TYPE`. Source: `backend/app/audit/services/audit_trail_service.py`.
- **Read flow (registry-backed):** validate `entity_type ∈ registry` → fetch audit rows → resolve the live entity via the registry/scope repository with `include_deleted` → if the live row is absent, resolve client context from immutable `metadata_json.client_record_id` → return `AUDIT.ENTITY_NOT_FOUND` (404) **only when neither a live entity nor usable historical audit metadata exists** → authorize the current ADVISOR/SECRETARY role → apply the sensitive-data hook → map to response. `entity_deleted` is set once on the envelope. Source: `backend/app/audit/services/audit_trail_service.py`.
- The read API accepts `page`/`page_size` plus optional `action`, `user_id` (→ `performed_by`), `created_after`/`created_before` (→ `performed_at`). Trail queries are scoped to the exact `(entity_type, entity_id)` pair, newest-first by `performed_at DESC, id DESC`. Source: `backend/app/audit/repositories/audit_entity_audit_log_repository.py`.
- Display reads **prefer the immutable `actor_display_name` snapshot** over the live-join `performed_by_name` (the latter tracks renames and is a fallback only). Source: `backend/app/audit/services/audit_trail_service.py`; frontend `AuditTrailTable` prefers `actor_display_name → performed_by_name → #id → —`.
- **Append-only repositories (Phase 2):** `EntityAuditLogRepository` + `UserAuditLogRepository` extend `AppendOnlyRepository`, not `BaseRepository` — they expose only `append`/`create` + read queries and have NO `update`/`delete`/`soft_delete`/`hard_delete`. Corrections are new rows. Source: `backend/app/common/repositories/append_only_repository.py`.
- **Same-transaction, fail-closed writes (Phase 2):** `EntityAuditWriter` validates every write (actor matrix + payload safety) and appends in the caller's transaction. An invalid payload or failed insert raises and rolls the domain mutation back; a rolled-back mutation leaves no orphan audit row. The old `actor_id is None` no-op is removed. Source: `backend/app/audit/services/audit_entity_audit_writer_service.py`.
- `EntityAuditWriter` normalizes payload snapshots (enums→`.value`, dates→ISO, decimals→str, bare strings→`{"value": ...}`) and stores the dict/list object directly in the JSONB column. Source: `backend/app/audit/services/audit_entity_audit_writer_service.py`.

## Registry, scope, actor matrix & sensitive-data policy (Phase 2)

- **`AuditEntityRegistry`** (`backend/app/audit/audit_entity_registry.py`) holds a pure descriptor per `entity_type` (model, resolution `strategy`, `sensitive` flag). It executes no SQL. `ALLOWED_READ_ENTITY_TYPES` is derived from it. `deadline_rule` is intentionally excluded (no per-row UI edit route; conditional per plan §6).
- **Scope resolution** runs in `AuditScopeRepository` (DB access only) and is interpreted by `AuditTrailService` into an `AuditScope` (`client_ids`, `firm_level`, `entity_deleted`, `resolved_from = live_table | audit_metadata`). It supports firm-level / one / many clients, soft-deleted (live row + `include_deleted`), and hard-deleted (live row gone → client_ids from `metadata_json.client_record_id`). Scope is context, NOT per-user authorization.
- **Actor validation matrix (§5a)** — fail-closed on every write: `actor_type ∈ {user, system, external_signer}`; `user` requires `performed_by`; `system`/`external_signer` require `performed_by = None` AND `actor_display_name`. For `user` rows `actor_display_name` is strongly encouraged but NOT fail-closed (the `performed_by` FK gives a read-time fallback); strict enforcement + universal name-threading is a tracked follow-up (see progress log). Source: `backend/app/audit/audit_write_policy.py`.
- **Fail-closed payload safety (§16)** — a per-`(entity_type, action)` policy (`ACTION_POLICIES`): a positive top-level field allowlist for `old_value`/`new_value` (bounded by the request/snapshot shapes, so document content / unexpected fields reject — they are in no allowlist), required + allowed `metadata_json` keys, and `metadata_json`-must-be-an-object (a list/scalar rejects, never silently skips). The action's `<entity_type>.` prefix must match the row's entity_type. Defense-in-depth: a recursive forbidden-key denylist (`password*`/`token*`/`secret*`/`signing*`/`*content*`/keys + raw bytes) and compact-UTF-8-JSON size caps (`old_value`/`new_value` 32 KiB each, `metadata_json` 16 KiB). An action with no registered policy is rejected. Source: `backend/app/audit/audit_write_policy.py`.
- **Sensitive-data hook** — sensitive entity types (`signature_request`) pass through a service-owned hook. Under the current two-role model both ADVISOR and SECRETARY preserve the same forensic fields; forbidden data is rejected at write time. There is no lower-privilege authenticated role, so no per-role redaction exists. Source: `backend/app/audit/services/audit_trail_service.py`.
- **`metadata_json.client_record_id`** is enriched on every existing client/business/charge/annual-report write (plus `business_id`/`tax_year` where relevant) so client-context reads and the §8b expression index work.

## Error codes

The audit module raises these `AUDIT.*` codes:

- `AUDIT.INVALID_ENTITY_TYPE` for unsupported read targets.
- `AUDIT.ENTITY_NOT_FOUND` (404) when neither a live entity nor usable historical audit metadata exists.
- `AUDIT.INVALID_ACTOR` (500) when a write violates the §5a actor matrix.
- `AUDIT.FORBIDDEN_FIELD` (500) for a forbidden/non-allowlisted payload field (§16).
- `AUDIT.PAYLOAD_TOO_LARGE` (500) when a normalized payload exceeds its size cap (§16).

Source: `backend/app/audit/services/audit_trail_service.py`, `backend/app/audit/audit_write_policy.py`.

Registry: `docs/backend/error-codes.md`.

## Known issues

- **§5a follow-up (deferred):** `actor_display_name` is fail-closed only for `system`/`external_signer`; for `user` rows it is encouraged but not enforced (the `performed_by` FK gives a read-time fallback). Audit rows written by internal machine paths (obligation orchestrator, freeze/close cascade, seed, excel import) therefore may carry no name snapshot and are not rename-stable. Strict `user → display required` + threading a display name through every such path is tracked for a later phase. See `docs/audit-refactor-progress.md` and `docs/audit-refactor-implementation-plan.md` §5a.

## Decisions (preserved)

Still-true decisions preserved from the shared historical reference `backend/docs/history-vs-timeline.md`:

1. `AuditTrail` is the accountability log for audited entity changes, separate from the operational `Timeline`. Source: `backend/docs/history-vs-timeline.md:18-22`.
2. The system should not dump the full audit stream into timeline views; timeline may use only selected high-value audit-derived events, while audit remains the complete record. Source: `backend/docs/history-vs-timeline.md:20-22`.
3. Generic audit inspection is registry-driven (Phase 2). `client`, `business`, `charge`, and `annual_report` are the families that currently *write* audit; the registry additionally declares the full §6 entity set so reads/scope are defined for every audited type as later phases add their writes. `ALLOWED_READ_ENTITY_TYPES` is derived from the registry. Source: `backend/app/audit/audit_entity_registry.py`.

## Future / planned

- A generic entity-audit UI can be reused for `client`, `business`, `charge`, and `annual_report`; this is described in the shared historical note but is not implemented in `backend/app/audit` itself. Source: `backend/docs/history-vs-timeline.md:24-29`.
- The timeline may later include selected audit-derived events, but not the complete audit stream. This is future intent only, not a current integration contract. Source: `backend/docs/history-vs-timeline.md:22`.

## Historical notes

No audit-specific `backend/docs/**` legacy file was found to archive. The only relevant legacy material was the shared conceptual reference `backend/docs/history-vs-timeline.md`, which was left unchanged because it is not audit-exclusive and also informs the timeline boundary.
