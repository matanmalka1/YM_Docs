# Audit Refactor — Progress Log

## Current Status

Status: Phase 0 completed. Baseline green. No blockers. Phase 1 not started.

Current phase:
- Phase 0 — Baseline, exact inventory, enum audit (Completed)

Next phase:
- Phase 1 — Schema add/alter only (migrations 1a→1b); not started

Phase 0 report:
- docs/audit-refactor-phase-0-report.md

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
| Phase 0 | Baseline, exact inventory, enum audit | Completed | Baseline green; exact inventory + matrices produced; no blockers | docs/audit-refactor-phase-0-report.md |
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
- Completed

Date:
- 2026-06-29

Goal:
- Run baseline checks and produce the exact pre-implementation inventory + binding matrices. No implementation.

Files changed:
- docs/audit-refactor-phase-0-report.md (created)
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
- pytest 1517 passed (user-confirmed). vulture pass. pyright 0/0/0. ruff format --check pass (915 files). ruff check pass. Scripts: migration 0, role 0 (217 routes), pagination 0 (31 endpoints), unused 4 (known external/manual), enums 0, schema 43 (informational). Health checks pass. OpenAPI in sync. Frontend lint/typecheck/tests/format/arch/arch:strict/knip pass.

Result:
- Baseline green. Exact inventory produced: 7 audit models, 57 audit write sites (47 → EntityAuditLog, 10 UserAuditLog), 9 read routes, 42 migration tables, 5 legacy tables to drop, 5 frontend consumer areas, 5 seed builders. Plus binding action matrix, AuditEntityRegistry/scope matrix, signature replacement inventory, timeline/dashboard inventory, actor availability inventory.

Important findings:
- Legacy audit tables own NO exclusive Postgres enum (action/event_type/actor_type/field_name are plain String). Only enum involved, annualreportstatus, is shared with annual_reports and must stay → cleanup migration drops ZERO enum types.
- json.dumps in EntityAuditWriter (audit_entity_audit_writer_service.py:143) + UserAuditLogRepository.create (:34); json.loads in AuditLogService._to_dict (:83) — must change with the JSONB switch (Phase 1).
- build_with_audit already receives items (no repo dependency) — supports the AuditTrailService-fed layering for the signature drawer.
- 5 missing-actor surfaces confirmed: advance_payment create + update, authority_contact add + update, permanent_document delete (all have route current_user available).

Decisions made:
- None new; confirmed plan assumptions. Open item for Phase 2: confirm SECRETARY has no audit-read restriction beyond existing route auth + client filtering (current finding: none).

Risks/blockers:
- No blockers. Risks carried: signature forensic parity, timeline duplicate events, JSON serializer/reader switch, Phase-1 NOT NULL sequencing, fail-safe downgrade, shared annualreportstatus enum, layering, frontend per-phase migration.

Next safe step:
- Phase 1 (schema add/alter only; migrations 1a→1b; serializer/reader changes; writer actor support). No legacy table drops in Phase 1.

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

### Phase 0 — Completed (2026-06-29)

- Baseline checks run; all green (pytest 1517, vulture, pyright 0/0/0, ruff format --check, ruff check, audit scripts [migration/role/pagination/enums 0; unused 4 + schema 43 = pre-existing informational], health, OpenAPI sync, frontend suite).
- Exact inventory + binding action matrix + AuditEntityRegistry/scope matrix + signature replacement inventory + timeline/dashboard inventory + actor availability inventory written to docs/audit-refactor-phase-0-report.md.
- Counts: 7 audit models; 57 audit write sites (47 → EntityAuditLog: record_* 14, vat 12, signature 9, binder 7, AR-status 3, intake-edit 2; UserAuditLog 10); 9 read routes; 42 migration tables; 5 legacy tables to drop; 5 frontend consumer areas; 5 seed builders.
- Key findings: legacy audit tables own ZERO exclusive Postgres enum (cleanup drops no enum; annualreportstatus shared with annual_reports → keep); json.dumps sites (audit_entity_audit_writer_service.py:143, user_audit_log_repository.py:34) + json.loads (user_audit_log_service.py:83) to change in Phase 1; build_with_audit already receives items (supports AuditTrailService-fed layering); 5 missing-actor surfaces confirmed (advance_payment create/update, authority_contact add/update, permanent_document delete).
- No blockers. Recommendation: Phase 1 safe to start. Phase 1 NOT started.
