# Audit Refactor — Progress Log

## Current Status

Status: Phase 1 COMPLETED (schema + JSON-object switch + actor threading + generic-audit contract sync). Phase 2 not started.

Current phase:
- Phase 1 — Schema + JSON switch + actor threading + generic-audit contract sync (Completed)

Next phase:
- Phase 2 — Writer helpers / repository / registry / authz / validation (not started)

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
| Phase 2 | Writer/repository + registry/authz | Not started | Add writer/repo APIs and AuditEntityRegistry | TBD |
| Phase 3 | Replace VAT audit | Not started | Move VAT audit to EntityAuditLog | TBD |
| Phase 4 | Replace AnnualReportStatusHistory | Not started | Move status + child actions to EntityAuditLog | TBD |
| Phase 5 | Replace binder lifecycle/intake logs | Not started | Move binder audit to EntityAuditLog | TBD |
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
