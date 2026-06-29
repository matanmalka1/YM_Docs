# Audit Refactor — Progress Log

## Current Status

Status: Phase 0 COMPLETED (two review rounds; all corrections applied, 4 open items decided). Phase 1 APPROVED, not yet started.

Current phase:
- Phase 0 — Baseline, exact inventory, enum audit (Completed)

Next phase:
- Phase 1 — Schema add/alter only (migrations 1a→1b); approved, not started

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
| Phase 0 | Baseline, exact inventory, enum audit | Completed | Two review rounds; counts 30/63/73, 29 legacy-model/38 test files, 11 missing-actor fns; 4 open items decided | docs/audit-refactor-phase-0-report.md |
| Phase 1 | Schema add/alter only | Not started | Alter EntityAuditLog/UserAuditLog only; no legacy drops | TBD |
| Phase 2 | Writer/repository + registry/authz | Not started | Add writer/repo APIs and AuditEntityRegistry | TBD |
| Phase 3 | Replace VAT audit | Not started | Move VAT audit to EntityAuditLog | TBD |
| Phase 4 | Replace AnnualReportStatusHistory | Not started | Move status + child actions to EntityAuditLog | TBD |
| Phase 5 | Replace binder lifecycle/intake logs | Not started | Move binder audit to EntityAuditLog | TBD |
| Phase 6 | Replace SignatureAuditEvent | Not started | Move signature audit and drawer trail to EntityAuditLog | TBD |
| Phase 7 | Rebuild timeline/dashboard | Not started | Single-source registry, no duplicates | TBD |
| Phase 8 | Add missing audit writes | Not started | Add remaining domain audit coverage | TBD |
| Phase 9 | Cleanup migration + seeds | Not started | Drop legacy tables, update seeds | TBD |
| Phase 10 | OpenAPI/frontend regen + verification | Not started | Regenerate contracts and update frontend | TBD |

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
