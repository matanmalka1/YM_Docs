# Audit Refactor — Progress Log

## Current Status

Status: Phase 5 COMPLETED (binder lifecycle + intake-edit audit moved to generic `EntityAuditLog` after Phase 4 moved annual-report status audit). Phase 6 not started.

Current phase:
- Phase 5 — Replace binder lifecycle/intake logs (Completed)

Next phase:
- Phase 6 — Replace SignatureAuditEvent (not started)

Phase 0 report:
- docs/audit-refactor-phase-0-report.md (revised twice; status COMPLETED)

Main plan file:
- docs/audit-refactor-implementation-plan.md

Progress log:
- docs/audit-refactor-progress.md

## Locked Architecture Decisions

- Final audit models:
  - EntityAuditLog
  - UserAuditLog

- EntityAuditLog records:
  - business mutations
  - selected evidence events such as signature_request.viewed/signed/declined, annual_report.submitted, charge.paid, binder.handed_over

- UserAuditLog records:
  - auth
  - security
  - admin-access events
  - user create/activate/deactivate/role-change

- Removed legacy audit models:
  - VatAuditLog
  - AnnualReportStatusHistory
  - BinderLifecycleLog
  - BinderIntakeEditLog
  - SignatureAuditEvent

- No dual-write.
- No backward compatibility.
- No legacy wrappers.
- No backfill.
- Do not rewrite the initial Alembic migration.
- Use staged migrations.
- Generic audit reads go through AuditEntityRegistry authorization.
- Signature audit drawer must continue working, re-sourced from EntityAuditLog.
- Audit rows require actor snapshot via actor_display_name.
- Sensitive data policy applies to all audit payloads.
- Audit writes must be append-only and transactional.

## Important Plan Corrections Already Applied

- Legacy tables are not dropped in Phase 1.
- Phase 1 only alters EntityAuditLog and UserAuditLog.
- Legacy tables are dropped after final consumers are removed.
- PostgreSQL is the migration round-trip target.
- SQLite JSONB round-trip is not claimed.
- UserAuditLog.metadata_json also migrates to JSONB.
- EntityAuditLog gets:
  - actor_type
  - actor_display_name
  - metadata_json
  - nullable performed_by
  - JSONB old_value/new_value
- UserAuditLog gets:
  - metadata_json JSONB
  - actor_display_name
  - target_display_name
- metadata_json->>'client_record_id' requires a PostgreSQL expression index.
- Annual-report child actions stay semantic.
- BinderIntakeEditService is preserved; only its legacy log dependency is replaced.
- Signature forensic metadata is per-action, not blanket-required.
- content_hash is captured when available; missing hash is flagged, not made a hard requirement unless domain validation changes.
- Append-only and same-transaction audit writes are required.

## Phase Summary Table

| Phase | Name | Status | Summary | Report |
|---|---|---|---|---|
| Pre-Phase 0 | Setup progress log | In progress | Create this context/progress file | docs/audit-refactor-progress.md |
| Phase 0 | Baseline, exact inventory, enum audit | Completed | Three review rounds; counts 30/63/73, 20 legacy-model-class/13 consumer/38 test files, 11 missing-actor fns; 4 open items decided | docs/audit-refactor-phase-0-report.md |
| Phase 1 | Schema + JSON switch + actor threading + generic-audit contract sync | Completed | entity_audit_logs/user_audit_logs only (no legacy drops): migrations 1a→1b (PG round-trip + fail-safe proven), serializer/reader JSON-object switch, threaded actor into 30 EntityAuditLog writers + UserAuditLog snapshots, regenerated generic-audit OpenAPI/types/frontend, docs/domains/audit.md | this file (Phase 1 section) |
| Phase 2 | Writer/repository + registry/authz | Completed | Append-only repos; writer record_action/record_external_action + §5a actor matrix + §16 validation (no-op removed); repo client-context queries; AuditEntityRegistry + resolve_scope (23 types) + role authz; envelope entity_deleted; metadata enrichment + namespaced actions; frontend synced | this file (Phase 2 section) |
| Phase 3 | Replace VAT audit | Completed | VAT writes/read UI moved to EntityAuditLog generic audit; legacy VAT audit route/repo/schema/seed removed; table kept for Phase 9 | this file (Phase 3 section) |
| Phase 4 | Replace AnnualReportStatusHistory | Completed | Annual-report status writes/read UI/timeline moved to EntityAuditLog generic audit; legacy AR audit route/repo/schema removed; table kept for Phase 9 | this file (Phase 4 section) |
| Phase 5 | Replace binder lifecycle/intake logs | Completed | Binder lifecycle + intake-edit audit moved to EntityAuditLog; timeline + dashboard binder sources repointed; legacy route/service/repos/schemas removed; frontend on generic route | this file (Phase 5 section) |
| Phase 6 | Replace SignatureAuditEvent | Not started | Move signature audit and drawer trail to EntityAuditLog | TBD |
| Phase 7 | Rebuild timeline/dashboard | Not started | Single-source registry, no duplicates | TBD |
| Phase 8 | Add missing audit writes | Not started | Add remaining domain audit coverage | TBD |
| Phase 9 | Cleanup migration + seeds | Not started | Drop legacy tables, update seeds | TBD |
| Phase 10 | Final full verification + contract sync | Not started | Final repo-wide contract sync + full verification (frontend consumers already migrated per-phase 1/3/4/5/6) | TBD |

## Current Files Created

- docs/audit-refactor-implementation-plan.md
- docs/audit-refactor-progress.md
- docs/audit-refactor-phase-0-report.md

## Current Files Not Yet Created

- (none pending for Phase 0)

## Known Risks

1. Signature audit forensic/legal parity.
2. Generic audit route authorization and sensitive data exposure.
3. Migration ordering and reversible downgrade.
4. Timeline duplicate events if registry is incomplete.
5. metadata_json.client_record_id discipline and index usage.
6. Actor display snapshot correctness.
7. Append-only and same-transaction enforcement.
8. Frontend generated types and deleted per-domain routes.

## Phase 0 — Baseline, exact inventory, enum audit

Status:
- Completed (two review rounds; corrections applied; 4 open items decided; Phase 1 approved)

Date:
- 2026-06-29

Goal:
- Run baseline checks and produce the exact pre-implementation inventory + binding matrices. No implementation.

Files changed:
- docs/audit-refactor-phase-0-report.md (created, then revised after review)
- docs/audit-refactor-progress.md (updated)

Production code changed:
- no

Migrations changed:
- no

Schemas/OpenAPI changed:
- no

Frontend changed:
- no

Tests/checks run:
- Run IN Phase 0: vulture (pass), pyright (0/0/0), ruff format --check (pass), ruff check (pass), audit scripts (migration 0, role 0/217, pagination 0/31, unused 4, enums 0, schema 43).
- External / user-confirmed, NOT run in Phase 0: pytest 1517, frontend suite, OpenAPI sync, health checks. The authenticated health check (login + /me) is a STATE MUTATION (writes last_login_at + a UserAuditLog login_success row) and is excluded from read-only Phase 0; future read-only health checks must be unauthenticated.

Result:
- BLOCKED. First draft had inaccurate counts and provenance; report revised. Corrected counts: 30 existing EntityAuditLog write sites (14 record_* + 13 direct append() in annual_reports + 3 charge via record_charge_status_audit), business-target 63, total 73 (was 47/57). 7 audit models, 9 read routes, 42 tables, 5 legacy tables, 5 frontend areas, 5 seed builders, 22 legacy-model files, 13 legacy-repo files, 29 audit test files.

Important findings:
- Counts were undercounted in the first draft (missed 13 direct EntityAuditWriter.append() + 3 charge helper sites). Corrected to 30/63/73.
- annual_reports DOUBLE-writes today: EntityAuditLog (13 append) AND AnnualReportStatusHistory (3 append_status_audit_entry) — collapse in Phase 4 without losing semantics.
- Cleanup migration drops ZERO enum types (annualreportstatus shared with annual_reports). Per-table cleanup DDL enumerated; only signature_audit_events has a timestamp index (first draft wrongly claimed all five did).
- json.dumps (audit_entity_audit_writer_service.py:143, user_audit_log_repository.py:34) + json.loads (user_audit_log_service.py:83) to change in Phase 1.
- Raw action strings exist (signature event_type/actor_type; binder field_name) and must become constants; VAT has its own ACTION_* set; two ACTION_STATUS_CHANGED definitions to namespace.
- scope_to_active_clients_stmt is a deleted-client FILTER, not per-user authorization; audit history must stay readable after delete (resolve scope with include_deleted / audit metadata).
- actor_display_name source absent everywhere — decide Phase 2 (thread from current_user vs write-time users lookup).

Decisions made (4 open items resolved):
- (1) Signature forensic fields visible to BOTH ADVISOR and SECRETARY (preserve current behavior; stricter rule = separate permission change).
- (2) actor_display_name = thread current_user.full_name explicitly with actor_id; system/external-signer labels explicit.
- (3) Charges: create→charge.created, issue→charge.issued, pay→charge.paid, cancel→charge.canceled, delete→charge.deleted.
- (4) Annual reports: remove legacy AnnualReportStatusHistory writes in Phase 4; preserve create→annual_report.created, status→annual_report.status_changed, deadline→annual_report.deadline_updated, child→semantic actions.

Round-2 corrections: counts 30/63/73; legacy-model reference files 29 (was 22); audit test files 38 (was mislabelled 29); missing-actor functions corrected to 11 (added invoice attach [route lacks CurrentUser], task update/bulk_assign/delete, correspondence update_entry, reminder cancel_reminder); AR does NOT use ACTION_METADATA_UPDATED; charge.created/deleted are genuine; business.*/signature_request.* are shorthand not binding lists.

Risks/blockers:
- No blockers. Risks carried as in §15 of the report.

Next safe step:
- Phase 1 — schema add/alter (migrations 1a→1b) + serializer/reader changes + writer actor support (actor_display_name threaded). No legacy table drops.

## Phase 1 — Schema + JSON switch + actor threading + generic-audit contract sync

Status:
- Completed

Date:
- 2026-06-29

Goal:
- Convert the two surviving audit tables to the target shape (JSONB payloads + actor snapshots), switch serializer/readers off JSON-string encoding, thread actor identity into every existing EntityAuditLog write and the UserAuditLog auth/admin writers, and ship the forced generic-audit response-shape change end-to-end (OpenAPI + frontend). No legacy audit tables touched.

Files changed:
- Backend models: `app/audit/models/audit_entity_audit_log.py` (JSONB old/new/metadata via `JSON().with_variant(JSONB,"postgresql")`, `actor_type` NOT NULL default `"user"`, `actor_display_name`, nullable `performed_by`, perf indexes + dialect note on the §8b expression index), `app/users/models/user_audit_log.py` (metadata_json→JSONB, `actor_display_name`, `target_display_name`; no actor_type/old/new).
- Backend repos/writer: `app/audit/repositories/audit_entity_audit_log_repository.py` (append: actor_type/actor_display_name/metadata_json, performed_by nullable), `app/audit/services/audit_entity_audit_writer_service.py` (drop `json.dumps`; `_serialize_value` returns normalized object; actor params on append + record_*), `app/charges/charge_billing_audit.py`, `app/users/repositories/user_audit_log_repository.py` (dict metadata + display snapshots), `app/users/services/user_audit_log_service.py` (drop `json.loads`; display snapshots in `_to_dict`/`log`).
- Backend schemas: `app/audit/schemas/audit_entity_audit_log.py` (old/new/metadata_json as JSON objects + actor_type/actor_display_name, performed_by nullable), `app/users/schemas/user_management.py` (actor_display_name/target_display_name on `UserAuditLogResponse`).
- Backend reader: `app/audit/services/audit_trail_service.py` (null-safe performed_by collection; response carries snapshot; reads prefer snapshot).
- Actor threading — 30 EntityAuditLog write sites + their service/route call chains: clients (create/update/lifecycle + excel import), businesses (create/update/lifecycle + client-business facade), charges (create/issue/pay/cancel/delete + bulk), annual_reports (create/status/deadline/detail/annex/financial-lines/vat-import/delete). UserAuditLog snapshot capture: `user_auth_service.py`, `user_management_service.py` (+ their routes).
- Migrations: `alembic/versions/0002_audit_jsonb_actor.py` (1a) + `0003_audit_actor_type_notnull.py` (1b), numeric-prefixed per `docs/backend/migrations.md`.
- Seed: `app/seed/builders/users.py` (dict metadata + actor/target snapshots on UserAuditLog; actor_type + actor_display_name on seeded EntityAuditLog rows).
- Frontend: `src/types/generated.ts` (surgical audit-schema edits — full regen is unrelated version churn), `openapi.json` (regenerated; diff is 100% audit), `src/features/audit/api/contracts.ts` (JSON-object zod + nullable performed_by + actor fields), `src/features/audit/utils/auditFormatters.ts` (no JSON.parse; `AuditDiffInput` decouples the formatter from the full entry type), `AuditTrailTable.tsx` (optional `actor_display_name` + `actor_display_name → performed_by_name → #id → —`), `useEntityAuditTrailSection.ts` (skip null-actor rows), `src/features/timeline/normalize.ts` + `timeline/api/contracts.ts` (consume the decoupled formatter; change_old/change_new typed as JSON), audit feature barrels.
- New tests: `tests/audit/test_phase1_json_roundtrip.py` (dict round-trip both tables + response objects + snapshots), `tests/audit/test_phase1_migration_roundtrip.py` (Postgres-gated: clean upgrade→downgrade→re-upgrade + fail-safe refusal), `tests/audit/test_audit_endpoint.py::test_actor_display_name_snapshot_survives_user_rename`. Updated string-contract tests across audit/charges/clients/annual_reports to the JSON-object contract; updated bulk-billing + auto-populate fakes for the new `actor_name` kwarg.

Production code changed:
- yes

Migrations changed:
- yes (two new revisions; initial migration untouched)

Schemas/OpenAPI changed:
- yes (EntityAuditLogResponse old/new→objects + actor fields + nullable performed_by; UserAuditLogResponse + actor/target display)

Frontend changed:
- yes (generic-audit contracts/formatter/table/hook + timeline formatter adapter + generated.ts/openapi.json)

Tests/checks run:
- Backend: full `pytest` (re-run after review fixes), `ruff check` (pass), `ruff format --check` (pass), `pyright` (0/0/0), `vulture` (only pre-existing database.py event-hook findings), audit scripts — migration chain (linear, 3 files), role 217, pagination 31, enum sync clean. SQLite `create_all` builds both models. PostgreSQL up→down round-trip on a throwaway DB: upgrade initial→1a→1b clean; clean downgrade→re-upgrade restores JSONB; fail-safe downgrade with a NULL-performed_by row **refused atomically** with a clear message (no row deletion, schema unchanged).
- Frontend (verification only): typecheck, lint, test (59), format:check, arch:check, arch:check:strict, unused — all green.

Result:
- COMPLETED. Two-model target shape in place for `entity_audit_logs`/`user_audit_logs`; serializer/readers off JSON strings; actor snapshots threaded; generic-audit contract synced backend↔frontend; migrations round-trip cleanly with a fail-safe downgrade. No legacy audit table/model/repo/route, timeline behavior, dashboard, or signature drawer changed.

Important findings:
- Committed `openapi.json` was actually in sync for non-audit paths; the fresh export diff was 100% audit (78 lines). `generated.ts` full regen is ~42k lines of unrelated openapi-typescript version/formatting churn, so the audit type changes were applied surgically (per the OpenAPI/generated-drift memo).
- The generic audit formatter was coupled to the full `EntityAuditLogEntry`; introduced `AuditDiffInput` (old/new: unknown) so the timeline (a hidden consumer) compiles without a behavior change. Timeline `change_old/change_new` are now JSON values because the backend timeline builder passes EntityAuditLog payloads straight through.
- `performed_by_name` (live users join) is retained as a fallback but reads now prefer the immutable `actor_display_name`; a rename regression test proves the snapshot is stable while the live-join name follows renames.

Decisions made:
- Kept the model-level `actor_type` default `"user"` (Python + temporary server-default in 1a) even though every writer passes it via the writer's own default — it keeps direct-ORM/seed construction safe and SQLite `create_all` valid. 1b enforces NOT NULL and drops only the server_default. (Reviewer noted "explicit actor_type"; the writer always supplies it — the model default is a deliberate belt-and-suspenders, not a substitute.)
- The §8b PostgreSQL expression index lives only in migration 0002 (documented dialect-specific exception in the model) because a portable JSON `->>'…'` expression index isn't expressible for SQLite `create_all`.
- Migration files use the numeric-prefix convention (`0002_…`, `0003_…`); the hash-named initial migration is left untouched per the locked rule.

Risks/blockers:
- DEFERRED wiring (Phase 6): the signature auto-submit annual-report transition (`signature_request_service.py`) still uses the `_SYSTEM_USER_ID = 0` sentinel. Phase 2 builds/tests the system/external-signer writer API, but does not wire this path while legacy `AnnualReportStatusHistory.changed_by` still requires a user FK. The signature path is wired after its legacy dependency is removed/repointed in the staged replacement phases.
- Legacy per-domain audit (VAT/binders/AR-status/signature) still emit/read their own shapes; their frontend consumers are untouched and migrate in Phases 3–6.

Next safe step:
- Phase 2 — append-only audit repositories, full `EntityAuditWriter` helpers + §5a actor validation matrix + §16 fail-closed write validation, repository-backed `AuditEntityRegistry` + deleted-history scope resolution, current-role authorization (both ADVISOR/SECRETARY), sensitive-data hook, namespaced actions, envelope-level `entity_deleted`, and required `metadata_json` enrichment. Build/test the system-actor API; signature sentinel wiring remains Phase 6.

## Phase 2 — Writer helpers / repository / registry / authz / validation

Status:
- Completed

Date:
- 2026-06-29

Goal:
- Add the full `EntityAuditWriter` helper + fail-closed validation surface, append-only audit repositories, repository-backed `AuditEntityRegistry` + scope resolution + current-role authorization, the read-flow rework (envelope `entity_deleted`, deleted-history readability), `metadata_json` enrichment + namespaced actions on the existing audited writes, and the forced frontend/OpenAPI sync. No legacy audit table/model/route touched; no new domains audited; no migration.

Files changed:
- Backend (new): `app/common/repositories/append_only_repository.py` (append-only base), `app/audit/audit_write_policy.py` (§5a actor matrix + §16 forbidden-key/metadata-allowlist/size-cap validation), `app/audit/audit_entity_registry.py` (23-type registry + `ScopeStrategy` + `allowed_read_entity_types`), `app/audit/audit_scope.py` (`AuditScope`/`EntityScopeResolution`), `app/audit/repositories/audit_scope_repository.py` (existence/scope DB resolution).
- Backend (changed): `app/audit/repositories/audit_entity_audit_log_repository.py` (now `AppendOnlyRepository`; added `list_by_entity`/`list_by_entities`/`list_for_client_context` (§8b)/`list_recent_activity`), `app/users/repositories/user_audit_log_repository.py` (`AppendOnlyRepository`; dropped dead `include_deleted`), `app/audit/services/audit_entity_audit_writer_service.py` (`record_action`/`record_external_action`, namespace via `entity_action`, validation, no-op removed), `app/audit/services/audit_trail_service.py` (registry orchestration, `resolve_scope`, role authz, sensitive hook, envelope mapping), `app/audit/audit_constants.py` (full `ENTITY_*` set, `entity_action`, pre-namespaced charge/AR actions, `NOTE_ENTITY_TYPE_CHANGED`), `app/audit/schemas/audit_entity_audit_log.py` (envelope `entity_deleted`), `app/audit/api/audit_routes.py` (passes `current_user`), `app/core/error_codes.py` (+`AUDIT.INVALID_ACTOR`/`FORBIDDEN_FIELD`/`PAYLOAD_TOO_LARGE`).
- Backend metadata enrichment + namespacing on existing writes: clients (create/update/entity-type-change/delete/restore), businesses (create/update/lifecycle delete+restore — client resolved via `get_by_legal_entity_id`), charges (`charge_billing_audit.py` + `charge_billing_service.py` create/issue/pay/cancel/delete), annual_reports (create/status/deadline/detail/annex/financial-line/vat-import/delete). Timeline/dashboard forced label updates: `app/timeline/timeline_audit_aggregator.py` (`_DEDUP_ACTIONS` namespaced), `app/dashboard/services/dashboard_recent_activity_service.py` (flat namespaced label/activity maps).
- Frontend: `src/features/audit/api/contracts.ts` (+`entity_deleted`), `src/features/audit/constants.ts` (namespaced `AUDIT_ACTION_LABELS` built programmatically + `AUDIT_ACTIONS_BY_ENTITY_TYPE` full namespaced values, incl. `client.entity_type_changed`), `src/features/audit/utils/auditFormatters.ts` (verb-suffix fallback). `backend/openapi.json` regenerated (audit-only). `src/types/generated.ts` regenerated **deterministically** via the pinned generator (see review-fix below) — `entity_deleted: boolean` + the pre-existing `date_from/date_to` drift; `package.json` `gen:types` now runs the pinned local `openapi-typescript@7.13.0` devDependency + `prettier --write` (no `npx --yes` unpinned).
- Tests (new): `tests/audit/test_phase2_registry_authz.py` (401; both roles; 404 only with no live+no history; soft/hard-deleted readable+flagged; both roles same forensic; actor matrix valid + every invalid combo rolls back; §16 forbidden/non-allowlisted/oversized; append-only mutators absent; atomicity savepoint rollback). Updated: `tests/audit/test_entity_audit_writer.py`, `test_audit_endpoint.py`, `test_audit_endpoint_filters.py`, `test_phase1_json_roundtrip.py`, `tests/charges/service/test_billing_audit.py`, `tests/clients/service/test_entity_type_change_guard.py`, plus 23 fallout fixes across annual_reports/businesses/charges/invoices/clients-repo/timeline tests (thread `actor_id`/namespaced actions where they relied on the removed no-op or bare actions).

Production code changed:
- yes

Migrations changed:
- no (Phase-1 schema is the target; all Phase-2 changes are response/validation/query/registry level on existing columns; `entity_deleted` is response-only)

Schemas/OpenAPI changed:
- yes (`EntityAuditTrailResponse.entity_deleted`; audit-only OpenAPI diff)

Frontend changed:
- yes (contracts/constants/formatter + surgical generated.ts)

Tests/checks run:
- Backend static: `ruff check` pass, `ruff format --check` pass (formatted the 7 authored files), `pyright` 0/0/0, `vulture` clean, audit scripts — migration chain linear, role 217 protected, pagination 31, enum sync clean, unused-routes 4 (pre-existing baseline), dump_schema informational. SQLite `create_all` builds. No migration added → no PostgreSQL round-trip needed this phase.
- Backend `pytest`: user ran the full suite; the only failures were 5 tests whose fixes landed after that run started — re-running those 5 individually passes. Net: full suite green (1537 + those 5).
- Frontend: typecheck, lint, `test` (59), `format:check`, `arch:check`, `arch:check:strict`, `unused` (knip) — all green. `gen:types` confirmed the known ~21k-line version/format churn (pre-existing drift memo); restored canonical `generated.ts` and applied the audit-only `entity_deleted` delta surgically.

Result:
- COMPLETED. Generic audit reads are registry-authorized with deleted-history readability and an envelope `entity_deleted`; audit repositories are append-only; writes are fail-closed (actor matrix + §16) and transactional; existing audited writes carry namespaced actions + `metadata_json.client_record_id`; frontend/OpenAPI synced. No legacy audit table/model/repo/route, signature service/drawer, dashboard architecture, or timeline source architecture changed.

Important findings:
- Strict §5a (`user → actor_display_name` required, fail-closed) broke ~108 existing user+test flows because display names are not threaded through internal orchestration/cascade/seed/excel paths (the obligation orchestrator passes `actor_name or ""`). Surfaced as a decision; user chose the narrow option.
- The committed `openapi.json` was in sync for non-audit paths; the audit export diff was exactly +5 lines (`entity_deleted`). `gen:types` still churns ~21k unrelated lines (openapi-typescript version/format), so the surgical-edit workflow from Phase 1 was reused.
- `deadline_rule` has no per-row UI edit route (only list + firm-wide bootstrap) → excluded from the registry (plan §6 "conditional"; documented).

Decisions made:
- **Actor matrix (narrow, recommended option chosen by user):** structural invariants stay fail-closed for all actor types (`actor_type` valid; `user ⇒ performed_by`; `system`/`external_signer ⇒ performed_by NULL + display required). For `user` rows `actor_display_name` is strongly encouraged but NOT fail-closed — the `performed_by` FK gives a read-time name fallback, so a missing snapshot degrades gracefully (not rename-stable) rather than rolling back the mutation. **Follow-up:** strict `user → display required` + threading a display name through every internal orchestration/cascade/seed/excel path that writes audit (tracked for a later phase, e.g. Phase 8 when those domains' writes are built out).
- **§16 policy (revised after review):** a full per-`(entity_type, action)` policy in `audit_write_policy.py` — positive top-level value-field allowlist for `old_value`/`new_value` (bounded by request/snapshot shapes, so `document_content`/unexpected fields reject), required + allowed `metadata_json` keys (`metadata_json` must be an object — a list/scalar rejects, never silently skipped), action↔entity_type namespace match, plus the recursive forbidden-key/raw-bytes denylist and size caps. (The earlier metadata-only interpretation was rejected in review and replaced with this.)
- **Action namespacing:** generic verbs are composed `<entity_type>.<verb>` by the writer's `record_*` helpers; charge/AR-child actions are pre-namespaced constants. `client.entity_type_changed` is a first-class semantic action (revised after review — was previously a `note` on a `client.updated` row); the back-compat alias was removed.
- **Append-only:** Option A (no `BaseRepository` inheritance) via `AppendOnlyRepository`. Now an architecture rule in `docs/backend/architecture.md`.
- **Scope is not authorization:** both roles read all; `client_ids` is contextual, so resolver precision is best-effort for polymorphic types (note/reminder) — existence + `entity_deleted` + 404 logic are the load-bearing outputs.

Risks/blockers:
- Internal-path `user` audit rows (orchestrator/cascade/seed) carry no `actor_display_name` snapshot, so they are not rename-stable (display falls back to the live `performed_by` join). Closed by the §5a follow-up above.
- Signature auto-submit `_SYSTEM_USER_ID=0` wiring remains deferred to Phase 6 (legacy `AnnualReportStatusHistory.changed_by` still requires a user FK). The system/external-signer writer API is built + tested but not wired to that path.

Next safe step:
- Phase 3 — Replace VAT audit end-to-end (writer repoint → `EntityAuditWriter`, reads via `AuditTrailService`, delete `VatAuditLogRepository` + schema + per-domain route + seed, migrate the VAT frontend audit surface). Legacy table drop stays in Phase 9.

## Phase Update Template

Every phase must append a new section below using this format:

## Phase X — <name>

Status:
- Not started / In progress / Completed / Blocked

Date:
- YYYY-MM-DD

Goal:
-

Files changed:
-

Production code changed:
- yes/no

Migrations changed:
- yes/no

Schemas/OpenAPI changed:
- yes/no

Frontend changed:
- yes/no

Tests/checks run:
-

Result:
-

Important findings:
-

Decisions made:
-

Risks/blockers:
-

Next safe step:
-

## Changelog

### Pre-Phase 0

- Created progress/context log.
- Main implementation plan already exists at docs/audit-refactor-implementation-plan.md.
- Phase 0 has not started yet.

### Pre-Phase 0 — Codex review #2

- Codex second review found remaining blockers.
- Phase 0 is still not started.
- Plan updated to address the blockers (docs/audit-refactor-implementation-plan.md):
  - Phases 3–7 made end-to-end: writer + reader + route/schema + timeline/dashboard/drawer consumer + tests repointed before deleting any legacy repo/schema/service; legacy tables still dropped only in the Phase 9 cleanup migration. AnnualReportStatusHistory (Phase 4) repoints timeline; BinderLifecycleLog (Phase 5) repoints timeline + dashboard; SignatureAuditEvent (Phase 6) repoints timeline reader + embedded audit_trail.
  - Authorization registry restricted to the existing model: ADVISOR/SECRETARY roles + active/deleted client filtering. Removed "admin/accountant-of-record"; new ownership/permission models are out of scope.
  - resolve_client_record_id replaced by resolve_scope() returning AuditScope(client_ids, firm_level, entity_deleted, resolved_from); supports firm-level / one / many clients and soft- and hard-deleted entities (hard-deleted history read via immutable audit metadata).
  - Append-only audit repositories: documented that BaseRepository exposes update/delete/hard_delete; chosen Option A (audit repos do not inherit BaseRepository; append+read only), Option B fallback (override to raise). Tests assert mutation methods unavailable.
  - §16 sensitive-data writes are fail-closed: unknown fields fail validation and roll back; oversized payloads fail unless an approved summary contract exists; redaction is read-time only.
  - Signature content_hash: removed the follow-up-task workflow; missing hash on signed → metadata_json.content_hash_missing=true; no new queue/task introduced.
  - Signature drawer decision fixed: keep SignatureRequestWithAuditResponse and the embedded audit_trail, re-sourced from EntityAuditLog; drawer is not required to call the generic endpoint; wrapper is NOT deleted (new embedded item schema defined).
  - Added actor validation matrix (§5a): user→performed_by+display required; system/external_signer→performed_by null + display required; unknown actor_type fails; invalid combos roll back.
  - Phase 0 must produce a binding mutation→entity_type→action→old/new→metadata→actor matrix covering all audited domains (authority_contacts, notes, binder_handovers, reminders fired, documents, notifications, AR child actions); no implementation from the partial list.
  - Model JSON columns use JSON().with_variant(JSONB,"postgresql") for SQLite create_all safety; migrations stay PostgreSQL-specific with USING casts.
  - Phase-1 downgrade documented: JSONB→Text casts; performed_by stays nullable on downgrade (NOT NULL not safely restorable; rows not destroyed).
  - Phase 6 acceptance is backend-only (response tests, OpenAPI impact, embedded trail re-sourced); frontend drawer verification moved to Phase 10.
  - Verification commands: ruff format --check (not mutating ruff format); frontend lint/typecheck/test/format:check/arch:check/arch:check:strict/unused; "fix" removed.
  - Documentation ownership: primary audit docs in docs/domains/audit.md; docs/backend/architecture.md updated only if an architecture rule changes.
- Next safe step remains Phase 0 only — and only after this plan correction.

### Pre-Phase 0 — Codex review #3

- Codex review #3 found 5 remaining blockers.
- Phase 0 is still not started.
- Plan updated (docs/audit-refactor-implementation-plan.md) for:
  - Safe Phase 1 sequencing — split into migration 1a (additive schema + serializer/reader + writers passing actor_type/actor_display_name; actor_type nullable with temp server_default) then 1b (enforce actor_type NOT NULL + drop default) so no phase leaves existing audit writes failing.
  - JSON serializer/readers — EntityAuditWriter stops json.dumps and writes dict/list directly; UserAuditLogRepository writes dict; AuditLogService/AuditTrailService/schemas stop json.loads and treat old_value/new_value/metadata_json as JSON objects; added JSON-object round-trip tests.
  - Frontend-per-phase migration — Phases 3–6 each migrate their frontend consumer + regen OpenAPI/generated types + run frontend checks for that surface (no legacy wrappers/temporary routes); Phase 10 is the final full verification/contract sync, not the first frontend migration. Phase 6 updates the SignatureRequestAuditDrawer contract in-phase since the embedded audit_trail item shape changes.
  - Current auth model only — removed "wrong owner" tests/wording; Phase 0 produces a binding authorization matrix (entity_type → allowed_roles → scope_policy → deleted_entity_policy → forensic/sensitive visibility) using ADVISOR/SECRETARY + existing active/deleted filtering; tests cover 401/403-by-role/active-deleted-preserved/redaction-per-matrix/hard-deleted-from-audit-metadata; no invented owner/accountant permissions.
  - Layering — repositories do DB access only; AuditTrailService owns registry lookup, resolve_scope, authorization, redaction, response mapping; routes call AuditTrailService; the signature response builder receives already-fetched audit items via AuditTrailService and does not query EntityAuditLogRepository directly (§3a/§4a/§12 updated).
  - Downgrade behavior — Option A clean/fail-safe: Phase-1 downgrade asserts no performed_by IS NULL rows before restoring NOT NULL and fails with a clear message otherwise (no silent asymmetric schema change, no row destruction); JSONB→Text casts defined.
- Next safe step remains Phase 0 only — and only after this correction.

### Phase 0 — Run, then BLOCKED by review (2026-06-29)

- Baseline checks run in Phase 0 (vulture, pyright 0/0/0, ruff format --check, ruff check, audit scripts) green; pytest/frontend/OpenAPI/health are external/user-confirmed (NOT run in Phase 0).
- First-draft report produced, then a Phase 0 review found gaps → report **revised** and status set to **BLOCKED**.
- Corrections applied to docs/audit-refactor-phase-0-report.md:
  - Write-site counts corrected: 30 existing EntityAuditLog sites (14 record_* + 13 direct EntityAuditWriter.append() in annual_reports + 3 charge via record_charge_status_audit); business-target 63; total 73 (was 47/57).
  - Baseline provenance made explicit (which checks ran in Phase 0 vs external/user-confirmed).
  - Disclosed health-check state mutation: authenticated login writes last_login_at + a UserAuditLog login_success row → not read-only; future read-only health checks must be unauthenticated.
  - Exact per-table cleanup DDL (columns/indexes/FK/PK) for the 5 legacy tables; corrected the false "every legacy table has a timestamp index" (only signature_audit_events does).
  - Action matrix now lists persisted names + missing actions (client.entity_type_changed, charge.created/deleted, annual_report.deadline_updated, annex_line_added) + a raw action/entity string inventory (signature/binder raw strings; VAT ACTION_* set).
  - Registry matrix un-grouped (one row per entity_type); clarified scope_to_active_clients_stmt is a deleted-client filter not per-user authz; reconciled deleted-history readability; per-role visibility encoded (only open item: signature forensic visibility per role).
  - Full actor inventory (every mutation + actor_display_name source, which is currently absent everywhere); added legal_entity/person/link actor-threading need.
  - Full legacy-model (22), legacy-repo (13), and audit test (29) reference lists added.
- Four open items flagged for sign-off: signature forensic visibility per role; actor_display_name sourcing; charge.created/deleted reconciliation; AR double-write collapse.
- Phase 1 NOT started; blocked until the revised report is accepted.

### Phase 0 — COMPLETED after review round 2 (2026-06-29)

- Round-2 review found remaining factual errors; corrected in docs/audit-refactor-phase-0-report.md:
  - Reference counts fixed: legacy-model reference files = 29 (was 22, now includes module-name imports + consumers); audit test files = 38 (the list already had 38 paths, mislabelled 29).
  - Actor inventory corrected: "five missing surfaces" was wrong. Confirmed-missing actor now = 11 functions: advance_payment create + update, authority_contact add + update, permanent_document delete, invoice attach/create (route also lacks CurrentUser), task update, task bulk_assign, task delete, correspondence update_entry, reminder cancel_reminder (+ legal_entity/person/link threading).
  - Action matrix corrected: charge.created and charge.deleted are genuine actions (not issue/paid aliases); annual_reports does NOT use ACTION_METADATA_UPDATED (that is VAT-only); deadline → annual_report.deadline_updated; business.*/signature_request.* labelled shorthand, not binding persisted lists.
- Four open items DECIDED: signature forensic visible to both roles; actor_display_name threaded from current_user.full_name with actor_id; charge action names (created/issued/paid/canceled/deleted); AR legacy history removed in Phase 4 with create/status_changed/deadline_updated/child semantic actions preserved.
- Status: Phase 0 COMPLETED. Phase 1 APPROVED. Phase 1 not yet started.

### Plan sync before Phase 1 (2026-06-29)

- Implementation plan synchronized so Phase 1 is internally consistent (docs/audit-refactor-implementation-plan.md):
  - Phase 1 now explicitly includes the generic-audit contract sync: because EntityAuditLogResponse.old_value/new_value change string→JSON object, openapi.json + generated.ts + src/features/audit/ contracts/formatters are regenerated/updated and frontend audit checks run IN Phase 1 (not Phase 2/10). §14 row updated accordingly.
  - Actor threading clarified to happen in Phase 1 (not Phase 2): minimal writer actor support + thread current_user.full_name + actor_type into all 30 existing EntityAuditLog write sites BEFORE migration 1b enforces actor_type NOT NULL. Phase 2 keeps the full EntityAuditWriter helpers / validation matrix / registry / authz / metadata enrichment.
  - Fixed stale statements: §15 Phase 1 no longer says "performed_by stays nullable on downgrade" — it now uses the fail-safe Option A downgrade (assert no performed_by IS NULL, else fail). §4a line no longer says signature frontend regenerates in Phase 10 — drawer + generated types migrate in Phase 6, with Phase 10 only the final full sync.
  - Phase 0 report count scoping fixed: legacy-model class-reference files = 20 (word-boundary grep), legacy repo/schema/consumer files = 13, test files = 38; the earlier single "29" label that didn't match its list is replaced by two exactly-counted sets.
- Phase 1 APPROVED post-sync. Still not started.

### Phase 1 — COMPLETED (2026-06-29)

- Implemented the two-model target shape for `entity_audit_logs` + `user_audit_logs` only (no legacy drops): JSONB old/new/metadata via `JSON().with_variant(JSONB,"postgresql")`, `actor_type` (NOT NULL, default "user") + `actor_display_name` + nullable `performed_by` on EntityAuditLog; JSONB `metadata_json` + `actor_display_name` + `target_display_name` on UserAuditLog (no actor_type/old/new).
- Serializer/reader switch: dropped `json.dumps` in `EntityAuditWriter._serialize_value` + `UserAuditLogRepository.create` and `json.loads` in `AuditLogService._to_dict`; `EntityAuditLogResponse`/`UserAuditLogResponse` expose JSON objects + the snapshot fields.
- Threaded `current_user.full_name` (+ `actor_type` via writer default) into all 30 existing EntityAuditLog write sites and their service/route call chains; UserAuditLog auth/admin writers capture actor/target display snapshots.
- Migrations `0002_audit_jsonb_actor` (1a) + `0003_audit_actor_type_notnull` (1b); PostgreSQL round-trip proven on a throwaway DB incl. clean upgrade→downgrade→re-upgrade and an **atomic fail-safe downgrade refusal** when NULL-performed_by rows exist; SQLite `create_all` validated.
- Generic-audit contract sync: regenerated `openapi.json` (audit-only diff) + surgically updated `src/types/generated.ts`; migrated `src/features/audit/` contracts/formatter/table/hook and the timeline formatter adapter (decoupled via `AuditDiffInput`); full frontend verification green.
- Seed builders updated to dict metadata + actor/target snapshots. Docs: `docs/domains/audit.md` updated (two-model shapes, actor-snapshot + JSON-object conventions, index/read-preference notes).
- Review round applied: numeric-prefix migration filenames; seed JSON-scalar→object fix; model docstring path + expression-index dialect note; clean-round-trip + rename-snapshot regression tests added; system-actor sentinel (signature auto-submit) documented as deferred to Phase 2/6.
- Backend full `pytest` green; `ruff check`/`ruff format --check`/`pyright`/audit scripts green. Phase 2 not started.

### Plan sync round 3 (2026-06-29)

- Separated EntityAuditLog vs UserAuditLog in the Phase-1 serializer/reader spec: UserAuditLogResponse has NO old_value/new_value and NO actor_type; it exposes `metadata` as a JSON object + actor_display_name/target_display_name snapshots. EntityAuditLog gets actor_type, actor_display_name, JSON old/new/metadata. Phase 1 also updates existing UserAuditLog writers (user_auth_service, user_management_service, password-reset) to capture the display-name snapshots (no actor_type).
- Phase 0 report: generic-audit frontend update moved from "Phase 2/10" to Phase 1 (the old_value/new_value string→object change ships with its frontend).
- Progress log: phase table corrected — Phase 0 legacy count now 20 legacy-model-class files (+13 consumer files), not 29; Phase 1 row expanded; Phase 10 relabelled "final full verification + contract sync" (frontend consumers already migrated per-phase), not a frontend-update step.
- Added docs/domains/audit.md update to Phase 1 (documentation-ownership rule: primary audit docs there; architecture.md only if a rule changes).
- Phase 1 spans project root (backend + frontend + docs), not backend-only.

### Phase 2 prompt synchronization (2026-06-29)

- Phase 2 execution prompt and source docs synchronized before implementation:
  - Fixed authorization to the real model: both ADVISOR and SECRETARY are allowed; no 403-by-role, owner/accountant, or per-role-redaction cases are invented. Audit reads bypass the active-client listing filter.
  - Defined hard-delete flow and `entity_deleted` as one envelope-level `EntityAuditTrailResponse` field; 404 only when neither live entity nor usable historical metadata exists.
  - Registry DB access is repository-owned; `AuditTrailService` orchestrates but does not execute SQL. Added the missing `correspondence` registry row.
  - Locked normalized compact-JSON limits: old/new 32 KiB each, metadata 16 KiB; no Phase-2 summary exception.
  - Phase 2 builds/tests system-actor support but defers signature auto-submit wiring to Phase 6 because legacy annual-report status history still requires a user FK.
  - Namespaced actions explicitly include the required audit frontend and narrow timeline label/test updates; no timeline source repointing or dedup redesign.
  - OpenAPI + canonical `npm run gen:types` are mandatory; manual generated-type edits are forbidden. The append-only audit-repository rule requires a Phase-2 update to `docs/backend/architecture.md`.
- Phase 2 remains not started. Revised execution prompt approved after final review.

### Phase 2 — COMPLETED (2026-06-29)

- Implemented the full Phase-2 surface (see the "Phase 2" section above for files/checks): append-only audit repositories (`AppendOnlyRepository`, no `BaseRepository` mutation surface — now an architecture rule); `EntityAuditWriter` `record_action`/`record_external_action` + §5a actor matrix + §16 fail-closed payload validation with the `actor_id is None` no-op removed; repository client-context queries (`list_by_entity`/`list_by_entities`/`list_for_client_context` §8b/`list_recent_activity`); repository-backed `AuditEntityRegistry` (23 entity types, `deadline_rule` excluded) + `resolve_scope` (firm/one/multi-client, soft + hard-delete-from-metadata) + current-role authorization; envelope-level `entity_deleted`; `metadata_json.client_record_id` enrichment + namespaced `<entity_type>.<verb>` actions across the existing client/business/charge/annual-report writes; forced timeline/dashboard label updates; frontend contracts/constants/formatter + surgical `generated.ts` + regenerated `openapi.json` (audit-only).
- **Decision recorded (actor matrix):** narrow option chosen by the user — structural invariants fail-closed for all actor types and `actor_display_name` required for system/external_signer; for `user` it is encouraged but not fail-closed (FK fallback). Follow-up logged: strict `user → display required` + threading display names through internal orchestration/cascade/seed/excel paths (deferred, likely Phase 8).
- **Decision recorded (§16 allowlist):** per-action field allowlist governs `metadata_json` (defined §8 contract); `old_value`/`new_value` carry the audited domain change, guarded by the recursive forbidden-key denylist + size caps.
- Verification: backend `ruff`/`pyright` (0/0/0)/`vulture`/audit-scripts green, SQLite `create_all` builds, full `pytest` green (the user's full-suite run showed 5 stale failures whose fixes landed mid-run; re-running those 5 passes). Frontend typecheck/lint/test(59)/format/arch/arch:strict/knip green. No migration added.
- Phase 3 (Replace VAT audit) completed; later sections record Phase 4 and Phase 5 completion.

### Phase 2 — review-round fixes (2026-06-29)

A code review found two P1 and several P2 issues; all addressed:

- **P1 — bulk audit atomicity:** `BulkBillingService.bulk_action` caught `AppError` and continued while the per-charge status flush had already happened → a failed audit could leave a mutation to commit without its row (§17). Each item now runs in its own `db.begin_nested()` savepoint, so an audit failure rolls back that charge's mutation; other items are unaffected. (3 unit tests updated to use a real session.)
- **P1 — fail-closed policy bypasses:** replaced the metadata-only allowlist with a full per-`(entity_type, action)` policy (`audit_write_policy.py`): positive `old_value`/`new_value` field allowlist (so `document_content`/raw content reject — in no allowlist), required + allowed `metadata_json` keys, `metadata_json`-must-be-an-object (a list no longer silently skips), action↔entity_type namespace match, recursive forbidden-key/raw-bytes denylist, size caps.
- **P2 — `client.entity_type_changed`** is now a real semantic action via `record_action` (was `client.updated` + note); the back-compat alias in `audit_constants.py` was removed; frontend labels/filters updated.
- **P2 — historical scope:** scope is now resolved from ALL of the entity's audit rows (unfiltered `list_by_entity`), never the current filtered/paged view; hard-deleted history returns 404 unless it is firm-level, self-scoped, or carries a usable `client_record_id`. Real `note`/`reminder` resolvers added (resolve via the target's registry descriptor) instead of always-empty.
- **P2 — seed:** `seed/builders/users.py` now writes EntityAuditLog through `EntityAuditWriter` (namespaced actions + `metadata_json` + validation), backdating `performed_at` after.
- **P2 — child identity metadata:** AR income/expense writes carry `section` + `line_id`; annex writes carry `line_id` + `line_number` (§7a).
- **P2 — deterministic `generated.ts`:** the "21k churn" was a missing `prettier` step; with `gen + prettier` the diff is the 2 audit-relevant lines. Pinned `openapi-typescript@7.13.0` as a devDependency, `gen:types` now runs the local binary + `prettier --write`, and `generated.ts` was regenerated deterministically (`entity_deleted: boolean` required, matching the generator; the surgical optional `?` was wrong).
- **Policy gaps caught by production-path tests (now fixed):** added `annual_report.deleted` policy; allowlisted `advance_rate_updated_at` (server-stamped, audited on `client.updated`) and `source_vat_categories` (vat-import expense provenance).
- Verification after fixes: `tests/charges tests/clients tests/businesses tests/annual_reports tests/audit` → 324 passed, 1 skipped; `ruff`/`pyright` green. Audit test files updated by a second pass (Codex) for the required-metadata policy: 48 passed.

### Phase 3 — COMPLETED (2026-06-30)

Goal:
- Replace VAT audit end-to-end: move VAT audit writes/reads off the legacy `VatAuditLog` repository/schema/route and onto generic `EntityAuditLog`, including the VAT frontend History tab. Keep the legacy table/model for the Phase 9 cleanup migration.

Files changed:
- Backend: `app/audit/audit_constants.py`, `app/audit/audit_write_policy.py`, VAT route/service/repository modules under `app/vat/`, new `app/vat/vat_audit.py`, `openapi.json`, and VAT/binder tests.
- Deleted backend code: `app/vat/repositories/vat_audit_log_repository.py`, `app/vat/schemas/vat_audit.py`, and the VAT-local `GET /vat/work-items/{id}/audit` route. `VatAuditLog` model/table remain until Phase 9.
- Seed: removed `create_vat_audit_logs` and its orchestrator call. Demo VAT rows are created directly, so they no longer fabricate legacy VAT audit rows.
- Frontend: VAT History moved from the deleted VAT audit API/hook/utils to the generic audit feature; VAT action/field labels added; `generated.ts` regenerated after OpenAPI export.
- Docs: `docs/domains/audit.md`, `docs/domains/vat.md`, `docs/flows/05-vat-work-item-creation.md`, and `docs/vat-history-product-followup.md` document the Phase 3 behavior and the product follow-up around invoice events no longer appearing in a strict work-item trail.

Implementation summary:
- Added fail-closed VAT action policies for `vat_work_item.created/status_changed/filed/amount_overridden/updated/deleted` and `vat_invoice.created/updated/amount_changed/deleted`.
- Promoted VAT audit actions to canonical namespaced constants in `app/audit/audit_constants.py`; removed the bare VAT `ACTION_STATUS_CHANGED` collision from VAT constants.
- Repointed VAT mutations to `EntityAuditWriter` with user actors and `actor_display_name` threaded from the route's current user.
- Work-item events anchor on `entity_type=vat_work_item`; invoice events anchor on `entity_type=vat_invoice`, with `metadata_json.vat_work_item_id` linking back to the owning work item.
- Generic reads are served by `/api/v1/audit/vat_work_item/{id}` and `/api/v1/audit/vat_invoice/{id}` through `AuditTrailService`.

Verification reported for Phase 3:
- Backend static/subset checks green: pyright 0/0/0, ruff check + format clean, vulture clean, audit scripts clean except the known pre-existing unused-routes reminders baseline, OpenAPI contract sync clean, SQLite `create_all` OK, seed reset OK.
- Targeted pytest: 317 passed, 1 skipped across VAT/binders/audit/timeline/dashboard/openapi/error-doc coverage, including new VAT write, read, and atomicity tests.
- Frontend VAT audit migration checks green: `npm run gen:types`, typecheck, lint, test, format, arch checks, and unused/knip.

Behavior note:
- Invoice add/update/delete/amount-change events now live on the `vat_invoice` audit trail. A VAT work-item History tab that reads only `/audit/vat_work_item/{id}` shows only work-item lifecycle events, not invoice events. This is the intended entity split. Product review of whether a combined VAT-period history view is needed is tracked in `docs/vat-history-product-followup.md`.

### Phase 4 — COMPLETED (2026-06-30)

Goal:
- Replace AnnualReportStatusHistory end-to-end for production status audit behavior: move annual-report status writes/reads off the legacy repository/schema/route and onto generic `EntityAuditLog`, including the annual-report frontend audit consumer and timeline status-event source. Keep the legacy table/model for the Phase 9 cleanup migration.

Files changed:
- Backend: annual-report status/create/query services and status route, annual-report repository composition, annual-report response schemas, timeline repository, targeted annual-report/timeline/OpenAPI tests, and `openapi.json`.
- Deleted backend code: `app/annual_reports/repositories/annual_report_status_audit_repository.py` and the annual-report-local `GET /annual-reports/{id}/audit` route/list schema. `AnnualReportStatusHistory` model/table remain until Phase 9.
- Frontend: annual-report timeline panel moved from the deleted annual-report audit API/query/component to the generic audit feature; annual-report audit endpoint/query/type/component dead code removed; `generated.ts` regenerated after OpenAPI export.
- Docs: `docs/domains/annual-reports.md`, `docs/domains/audit.md`, `docs/domains/timeline.md`, and `docs/flows/02-annual-report-status-transition.md` document the Phase 4 behavior.

Implementation summary:
- Removed the three production `append_status_audit_entry` write sites. Creation keeps `annual_report.created`; status transitions write `annual_report.status_changed`; deadline updates write `annual_report.deadline_updated`.
- Status-change audit rows preserve old/new status, note, `client_record_id`, `tax_year`, actor id, and `actor_display_name` from the route's current user.
- Timeline annual-report status events now read `EntityAuditLog` rows for `annual_report.status_changed`, preserving the existing `annual_report_status_changed` event shape and Hebrew labels.
- Generic reads are served by `/api/v1/audit/annual_report/{id}` through `AuditTrailService`; ADVISOR and SECRETARY can read the trail.

Verification reported for Phase 4:
- Targeted backend annual-report/timeline/OpenAPI tests cover generic annual-report audit reads, secretary access, old-route removal, timeline source migration, and status-mutation rollback on audit failure.
- Frontend annual-report audit migration checks regenerated OpenAPI types and ran the annual-report-relevant type/lint/test surface.

## Phase 5 — Replace binder lifecycle + intake logs

Status:
- Completed

Date:
- 2026-06-30

Goal:
- Move binder lifecycle and intake-edit audit off the legacy `BinderLifecycleLog` / `BinderIntakeEditLog` repositories and onto generic `EntityAuditLog`, end-to-end: writers, generic read route, the binder timeline reader, the dashboard recent-activity binder branch, and the binder frontend audit consumer. Keep `BinderIntakeEditService` (repoint its writes only). Delete the per-domain binder audit route + `BinderAuditService` + both log repositories + binder audit schemas. Legacy tables kept for the Phase-9 cleanup migration.

Files changed:
- Backend (new): `app/binders/binder_audit.py` (metadata + lifecycle/intake snapshot helpers, the Phase-3 `vat_audit.py` analog).
- Backend (audit shared, append-only): `app/audit/audit_constants.py` (binder lifecycle `ACTION_BINDER_*` + `ACTION_BINDER_INTAKE_UPDATED`), `app/audit/audit_write_policy.py` (per-action policies for the 7 binder lifecycle actions + `binder_intake.updated`; binder/intake metadata allowlists). The registry already declared `binder`/`binder_intake`/`binder_handover` (Phase 2) — no registry change.
- Backend (writers): `app/binders/services/binder_lifecycle_service.py` (dropped `BinderLifecycleLogRepository`; `_append_log`→`_record` via `EntityAuditWriter`; 7 `_append_log` sites → rich semantic `binder.*` actions; `log_initial_state` collapses the two `null→in_office`/`null→open` rows into one `binder.created`; threads `actor_display_name`). `app/binders/services/binder_intake_edit_service.py` (dropped `BinderIntakeEditLogRepository`; field changes → `binder_intake.updated` with `{"value": ...}` + field identity in metadata; service KEPT). Routes thread `current_user.full_name`: `binder_routes_receive_return.py` (6 lifecycle + bulk callers), `binder_routes_audit.py` (intake PATCH), `binder_handover_service.py` (grouped handover).
- Backend (deletions): `app/binders/services/binder_audit_service.py`, `app/binders/repositories/binder_lifecycle_log_repository.py`, `app/binders/repositories/binder_intake_edit_log_repository.py`, the `GET /binders/{id}/audit` route, and the `BinderAuditEntry`/`BinderAuditResponse` schemas (`app/binders/schemas/binder.py`). `get_binder_intakes` moved from `BinderAuditService` to `BinderIntakeService`; the `/intakes` GET + PATCH routes stay. `BinderLifecycleLog` / `BinderIntakeEditLog` **models + tables** kept (Phase 9). `model_registry.py` keeps the model imports (unchanged).
- Backend (timeline): `app/timeline/services/timeline_service.py` (binder branch only) — `_append_lifecycle_change_events` now reads `EntityAuditLog` `binder.*` via an action allowlist (`marked_full`/`reopened`/`marked_ready_for_handover`/`reverted_ready`), excluding `created`/`material_received`/`handed_over` (live builders cover reception/handover). Adapts each audit row to the existing `binder_lifecycle_change_event` builder via a `SimpleNamespace` view (builder file untouched). Dropped the unused `_status_str`.
- Backend (dashboard): `app/dashboard/services/dashboard_recent_activity_service.py` — dropped `BinderLifecycleLogRepository`/`BinderRepository`/`binder_rows`/`_serialize_binder`/`_binder_label`/negative ids; binder activity flows through the single `EntityAuditLog` stream; binder labels from the action-label table; client resolved from `metadata_json.client_record_id`; `binder.marked_ready_for_handover`/`binder.handed_over` → `done`.
- Frontend: `src/features/audit/api/contracts.ts` (`EntityAuditType` += `binder`,`binder_intake`), `src/features/audit/constants.ts` (binder verb labels + action lists + field labels, append-only), `src/features/binders/components/drawer/BinderAuditSection.tsx` (rewritten to render `EntityAuditTrailSection entityType="binder"` with binder field-value labels; same name/prop, drawer unchanged), removed dead `getAudit`/`BinderAuditResponse`/`BinderAuditEntry`/`binderAudit` endpoint/query-key/types and the now-unused `getBinder{Location,Capacity}StatusVariant` getters. `src/types/generated.ts` + `backend/openapi.json` regenerated (binder-audit-only diff).
- Tests: rewrote `tests/binders/api/test_binder_audit.py` (generic `/audit/binder/{id}` returns lifecycle events; ADVISOR + SECRETARY read; pagination; old route 404), `tests/binders/service/test_binder_lifecycle_service.py` (assert `EntityAuditLog` `binder.*` actions + new atomicity rollback test), `tests/binders/service/test_binder_intake_edit_service.py` (assert `binder_intake.updated` rows + metadata), `tests/binders/api/test_binders.py` (initial-state → one `binder.created`), `tests/dashboard/service/test_recent_activity_service.py` (binder label/activity-type), new `tests/timeline/service/test_timeline_binder_lifecycle.py` (binder events from EntityAuditLog). Adjusted `tests/core/test_openapi_audit_paths.py` (binder old-route expectation removed) and `tests/regression/test_readonly_endpoints_no_side_effects.py` (binder audit read via generic route).
- Docs: `docs/domains/binders.md` (audit section + endpoint table + intake-editing + decisions), `docs/domains/audit.md` (binder actions + Phase-5 generic-write set), `docs/domains/timeline.md` (binder source), `docs/domains/dashboard.md` (recent-activity binder source), `docs/flows/01`, `docs/flows/04`, `docs/flows/08`, and this progress log.

Production code changed:
- yes

Migrations changed:
- no (Phase 5 drops no tables; binder log tables drop in Phase 9)

Schemas/OpenAPI changed:
- yes (removed `BinderAuditResponse`/`BinderAuditEntry` + `GET /binders/{id}/audit`; binder lifecycle/intake now served by the existing generic `EntityAuditTrailResponse`)

Frontend changed:
- yes (generic audit consumer + contracts/constants + regenerated types)

Tests/checks run:
- Backend focused pytest: `tests/binders tests/timeline tests/dashboard tests/core/test_openapi_audit_paths.py tests/regression/test_readonly_endpoints_no_side_effects.py` → 127 passed. `ruff check` + `ruff format --check` clean on changed files; `pyright` 0/0/0 on changed app files (run with the project `pyrightconfig.json`); `rg` confirms no production refs to `BinderAuditService`/`BinderLifecycleLogRepository`/`BinderIntakeEditLogRepository`/old route/schemas/`_serialize_binder` remain. `export_openapi.py` + `check_contract_sync.py` → in sync (binder-audit-only diff, 191 lines removed).
- Frontend: `gen:types` (deterministic, 112-line binder-audit-only diff), `typecheck`, `lint`, `arch:check`, `arch:check:strict`, `test` (59), `unused`/knip — all green.

Result:
- COMPLETED. Binder lifecycle + intake-edit audit, the binder timeline lifecycle source, the dashboard recent-activity binder branch, and the binder frontend audit surface are all off the legacy logs and on `EntityAuditLog`. `BinderIntakeEditService` preserved (writes repointed). Legacy route/service/repos/schemas removed; legacy tables retained for Phase 9. No annual-report/signature/VAT code touched.

Important findings:
- The per-domain binder "audit" router (`binder_routes_audit.py`) also hosts the `/intakes` GET + intake PATCH routes (which stay). Deleting `BinderAuditService` required moving its `get_binder_intakes` reader into `BinderIntakeService` rather than deleting it outright.
- The pyright IDE diagnostics surfaced `responses=`/`isoformat` errors that the project's untracked `backend/pyrightconfig.json` suppresses; that config is not present in the Phase-5 git worktree. Copying it into the worktree (an untracked local file) reproduces the project's clean pyright result.
- Timeline binder lifecycle reading lived in `timeline_service.py` (not `timeline_repository.py`); the binder branch was repointed in place and the existing `binder_lifecycle_change_event` builder reused via a row-shape adapter, so the builder file stayed untouched.

Decisions made:
- `log_initial_state` collapses the two legacy initial lifecycle rows into a single `binder.created` row (both statuses in `new_value`). The legacy timeline already suppressed the `null→in_office` row, so timeline output is unaffected.
- Timeline binder lifecycle source excludes `binder.created`/`material_received`/`handed_over` (plan §4: reception + handover are live-builder territory). This also removes a prior duplicate where handover appeared as both `binder_handed_over` and a lifecycle-change event.
- Dashboard binder rows use the action-label table rather than the audit `note` (binder notes are operational reason strings, not display labels), preserving the prior dashboard label set.
- `actor_display_name` threaded from `current_user.full_name` on the lifecycle/intake/handover routes; internal callers (intake/onboarding `receive`, seed) pass none and rely on the `performed_by` FK fallback (§5a — `user` display is encouraged, not fail-closed).

Risks/blockers:
- Seed (`app/seed/builders/demo/binders.py`) still builds legacy `BinderLifecycleLog`/`BinderIntakeEditLog` rows directly; it is **out of Phase-5 scope** (seed is owned by Phase 9). The models/tables remain, so seed still imports and runs, but demo binder activity will not appear in the EntityAuditLog-backed dashboard/timeline until the seed is updated in Phase 9. Flagged for the Phase-9/integrator follow-up.
- `openapi.json` + `generated.ts` are regenerated-per-branch throwaways; expect a regenerate-after-merge by the integrator (Phase 4 touches the same generated files).

Next safe step:
- Integrator review + merge alongside Phase 4 (shared files: `audit_constants.py`/`audit_write_policy.py` append-only, `contracts.ts` union member add, timeline files different branches, generated files regenerated). Phase 6 (signature) is the next replacement phase.
