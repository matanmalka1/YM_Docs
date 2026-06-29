# Audit Refactor — Phase 0 Report (Baseline & Exact Inventory)

Date: 2026-06-29
Scope: **inventory + baseline only.** No production code, migration, model, schema, seed, generated file, frontend, or OpenAPI changed. Outputs: this report + `docs/audit-refactor-progress.md` update.

## 1. Executive summary

- Baseline is **green** across backend (pytest, vulture, pyright, ruff, audit scripts, health, OpenAPI sync) and frontend (lint/typecheck/test/format/arch/arch:strict/knip).
- **7 audit models** exist; target is **2** (`EntityAuditLog`, `UserAuditLog`). 5 legacy tables to merge & drop: `vat_audit_logs`, `annual_report_status_history`, `binder_lifecycle_logs`, `binder_intake_edit_logs`, `signature_audit_events`.
- Initial migration creates **42 tables** (confirmed).
- **Key enum finding (de-risks Phase 9):** none of the 5 legacy audit tables owns an exclusive Postgres enum. `action`/`event_type`/`actor_type`/`field_name` are plain `String`. The only enum involved, `annualreportstatus` (used by `annual_report_status_history.from_status/to_status`), is **shared** with the surviving `annual_reports.status` → **must NOT be dropped**. **Cleanup migration drops zero enum types.**
- **Key serialization finding (drives Phase 1):** `EntityAuditWriter._serialize_value` and `UserAuditLogRepository.create` write JSON **strings** via `json.dumps`; `AuditLogService._to_dict` reads via `json.loads`. The JSONB switch must be paired with removing these in the same step (plan §1b).
- **Layering finding (good):** `build_with_audit(request, audit_events)` already **receives** items, so the signature builder can be fed via `AuditTrailService` without a repo dependency.
- **No blockers found.** Recommendation: **Phase 1 is safe to start.**

## 2. Baseline check results

| Check | Command | Result |
|---|---|---|
| Tests | `APP_ENV=test JWT_SECRET=x pytest -q` | **1517 passed** (user-confirmed; background run here was cancelled per instruction) |
| Dead code | `vulture` | **pass** (rc=0) |
| Types | `pyright` | **0 errors, 0 warnings, 0 info** |
| Format | `ruff format --check .` | **pass** — 915 files already formatted |
| Lint | `ruff check .` | **All checks passed** |
| Migration chain | `scripts/audit/check_migration_chain.py` | **0 findings** — 1 migration file, linear |
| Role coverage | `scripts/audit/check_role_coverage.py` | **0 findings** — all 217 routes auth-protected |
| Pagination | `scripts/audit/check_missing_pagination.py` | **0 findings** — all 31 list endpoints paginated |
| Unused routes | `scripts/audit/check_unused_routes.py` | **4 findings** (known external/manual candidates; pre-existing, non-blocking) |
| Enum sync | `scripts/audit/check_enum_sync.py` | **0 findings** — no enum drift |
| Schema dump | `dump_schema.py` | **43 findings** (informational schema observations; pre-existing, non-blocking) |
| Health | `scripts/ops/health_check.py` | **pass** — /health, /info, login, /me all 200 |
| OpenAPI | `check_contract_sync.py` | **in sync** |
| Frontend | `lint, typecheck, tests, format, arch, arch:strict, knip` | **all pass** (user-confirmed) |

No check was modified to pass. The `unused (4)` and `schema (43)` findings are pre-existing informational outputs unrelated to audit and **do not block Phase 1**.

## 3. Exact counts

| Metric | Count | Note |
|---|---|---|
| Audit models | **7** | target 2 |
| Legacy tables to drop | **5** | §6 |
| Migration tables total | **42** | confirmed |
| Enum types to drop in cleanup | **0** | none exclusively owned (§7) |
| EntityAuditWriter `record_*` write sites | **14** | EntityAuditLog today |
| VAT `append_audit` write sites | **12** | |
| Signature `append_audit_event` write sites | **9** | |
| Binder `_append_log` write sites | **7** | |
| AR `append_status_audit_entry` write sites | **3** | |
| Binder-intake edit write sites | **2** | |
| **Business-target write sites (→ EntityAuditLog)** | **47** | 14+12+9+7+3+2 |
| `UserAuditLog.log` write sites | **10** | auth/admin |
| **Total audit write sites** | **57** | |
| Audit/history/timeline/dashboard/signature read routes | **9** | §5 |
| Seed builders writing legacy audit | **5** | §8 |
| Frontend audit consumers (feature areas) | **5** | audit, vatReports, binders, annualReports, signatureRequests |

> Counts above are this-scan figures; they supersede the provisional numbers in the plan and are the baseline for later phases.

## 4. Audit model inventory

| Model | File | Table | action field | Target |
|---|---|---|---|---|
| EntityAuditLog | `app/audit/models/audit_entity_audit_log.py` | `entity_audit_logs` | `action: str` | KEEP + upgrade |
| UserAuditLog | `app/users/models/user_audit_log.py` | `user_audit_logs` | `action: AuditAction` enum, `status: AuditStatus` enum | KEEP (JSONB + snapshots) |
| VatAuditLog | `app/vat/models/vat_audit_log.py` | `vat_audit_logs` | `action: str` (String) | DELETE → EntityAuditLog |
| AnnualReportStatusHistory | `app/annual_reports/models/annual_report_status_history.py` | `annual_report_status_history` | `from_status/to_status: AnnualReportStatus` (shared enum) | DELETE → EntityAuditLog |
| BinderLifecycleLog | `app/binders/models/binder_lifecycle_log.py` | `binder_lifecycle_logs` | `field_name/old_value/new_value: String` | DELETE → EntityAuditLog |
| BinderIntakeEditLog | `app/binders/models/binder_intake_edit_log.py` | `binder_intake_edit_logs` | `field_name: String` | DELETE → EntityAuditLog (keep `BinderIntakeEditService`) |
| SignatureAuditEvent | `app/signature_requests/models/signature_request.py` | `signature_audit_events` | `event_type/actor_type: String` | DELETE → EntityAuditLog |

## 5. Writer / reader / route inventory

**Writer sites** (non-test):
- EntityAuditWriter `record_*` (14): clients (`client_create_service.py:216`, `client_update_service.py:70,105`, `client_lifecycle_service.py:32,55`), businesses (`business_service.py:101,177`, `business_lifecycle_service.py:24,37`), charges (`charge_billing_service.py:78,193`), annual_reports (`annual_report_create_service.py:150`, `annual_report_status_service.py:161`, `annual_report_service.py:70`) + detail/annex/financial-line services.
- VAT `append_audit` (12): `vat_work_item_metadata.py:47,71`, `vat_data_entry_status_service.py:42,82`, `vat_data_entry_invoices_service.py:146,184`, `vat_data_entry_invoice_update_service.py:164`, `vat_data_entry_invoice_delete_service.py:54`, `vat_intake_service.py:116,150`, `vat_filing_service.py:98,122`.
- Signature `append_audit_event` (9): `signature_request_creation_service.py:114,122`, `signature_request_signer_service.py:32,53,88,126`, `signature_request_admin_service.py:45,68`, `signature_request_service.py:79`.
- Binder `_append_log` (7): in `binder_lifecycle_service.py`.
- AR `append_status_audit_entry` (3): in `annual_report_status_service.py` (+ create path).
- Binder-intake edit (2): `binder_intake_edit_service.py`.
- UserAuditLog `.log` (10): `user_auth_service.py` (auth), `user_management_service.py` (user mgmt), password-reset path.

**Read routes (9):**

| # | Route | File | Source today | After |
|---|---|---|---|---|
| 1 | `GET /audit/{entity_type}/{entity_id}` | `app/audit/api/audit_routes.py:18` | EntityAuditLog | STAYS + expand via registry |
| 2 | `GET /users/audit-logs` | `app/users/api/user_routes_audit.py:24` | UserAuditLog | STAYS |
| 3 | `GET /vat/work-items/{id}/audit` | `app/vat/api/vat_routes_queries.py:184` | VatAuditLog | DELETE → generic (Phase 3) |
| 4 | `GET /binders/{id}/audit` | `app/binders/api/binder_routes_audit.py:26` | BinderLifecycleLog | DELETE → generic (Phase 5) |
| 5 | `GET /annual-reports/{id}/audit` | `app/annual_reports/api/annual_report_routes_status.py:129` | AnnualReportStatusHistory | DELETE → generic (Phase 4) |
| 6 | `GET /binders/{id}/intakes` | `app/binders/api/binder_routes_audit.py:56` | BinderIntake | STAYS |
| 7 | `GET /clients/{id}/timeline` | `app/timeline/api/timeline_routes.py:19` | 9 categories | STAYS (re-pointed) |
| 8 | `GET /dashboard/overview` (`recent_activity`) | `app/dashboard/api/dashboard_routes_overview.py` | EntityAuditLog + BinderLifecycleLog | CHANGES (Phase 5/7) |
| 9 | `GET /signature-requests/{id}` (`audit_trail`) | `app/signature_requests/api/signature_request_routes_advisor.py:101` | SignatureAuditEvent (embedded) | CHANGES — keep wrapper, re-source (Phase 6) |

**Service readers:** `AuditTrailService.get_entity_audit_trail`, `VatReportService.get_audit_trail_enriched`, `BinderAuditService.get_binder_audit`, `AnnualReportService.get_report_audit`, `AuditLogService.list_logs`, timeline aggregator + `TimelineRepository`, dashboard `RecentActivityService.build`, signature `get_audit_trail` + `build_with_audit`.

**JSON serialization sites to change (Phase 1):** `app/audit/services/audit_entity_audit_writer_service.py:143` (`json.dumps`), `app/users/repositories/user_audit_log_repository.py:34` (`json.dumps`), `app/users/services/user_audit_log_service.py:83` (`json.loads`).

## 6. Frontend consumer inventory

| Backend change | Frontend consumers | Migrate in |
|---|---|---|
| Generic `/audit/{type}/{id}` gains fields | `src/features/audit/` — `api/audit.api.ts`, `api/contracts.ts`, `hooks/useEntityAuditTrail.ts`, `hooks/useEntityAuditTrailSection.ts`, `components/AuditTrailTable.tsx`, `components/EntityAuditTrailSection.tsx`; consumed by `features/businesses/.../BusinessDetailsCard.tsx`, `features/charges/.../ChargeDetailDrawer.tsx` | Phase 2/10 (additive) |
| `/vat/work-items/{id}/audit` deleted | `features/vatReports/api/endpoints.ts` (`vatWorkItemAudit`), `api/vatReports.api.ts` (`getAuditTrail`), `api/contracts.ts` (`VatAuditTrailResponse`), `hooks/useVatHistory.ts`, `components/detail/VatHistoryTab.tsx` | **Phase 3** |
| `/annual-reports/{id}/audit` deleted | `features/annualReports/components/statusTransition/StatusAuditTimeline.tsx` | **Phase 4** |
| `/binders/{id}/audit` deleted | `features/binders/` — `components/drawer/BinderAuditSection.tsx`, `BinderDetailDrawer.tsx`, `api/endpoints.ts`, `api/queryKeys.ts`, `api/binders.api.ts`, `types.ts` | **Phase 5** |
| `/signature-requests/{id}` `audit_trail` item shape changes | `features/signatureRequests/components/drawer/SignatureRequestAuditDrawer.tsx`, `api/contracts.ts`, `api/signatureRequests.api.ts` | **Phase 6** |
| Generated types | `src/types/generated.ts` — paths for routes 3/4/5, schemas `EntityAuditTrailResponse`, `VatAuditTrailResponse`, `SignatureAuditEventResponse` (embedded `audit_trail` at line ~7135) | per-phase + final Phase 10 |

## 7. Enum ownership table

| Enum | Used by (table.column) | Shared with surviving table? | Safe to drop in cleanup? | Reason |
|---|---|---|---|---|
| `annualreportstatus` | `annual_report_status_history.from_status/to_status` **and** `annual_reports.status` | **YES** (annual_reports) | **NO** | shared with surviving table; dropping would break `annual_reports` |
| `auditaction`, `auditstatus` | `user_audit_logs.action/status` | n/a (surviving table) | **NO** | belongs to surviving UserAuditLog |
| (vat audit) | `vat_audit_logs.action` = plain `String` | — | n/a | no enum to drop |
| (binder lifecycle) | `binder_lifecycle_logs.field_name/old/new` = `String` | — | n/a | no enum |
| (binder intake edit) | `binder_intake_edit_logs.field_name` = `String` | — | n/a | no enum |
| (signature audit) | `signature_audit_events.event_type/actor_type` = `String` | — | n/a | no enum (SignatureRequest owns `signaturerequeststatus`/`type`, but those stay with the surviving table) |

**Conclusion:** the Phase 9 cleanup migration performs `DROP TABLE` on the 5 legacy tables and **drops no enum type**. `annualreportstatus` stays.

## 8. Table inventory

42 tables (confirmed). **Five legacy audit tables to drop (Phase 9):** `vat_audit_logs`, `annual_report_status_history`, `binder_lifecycle_logs`, `binder_intake_edit_logs`, `signature_audit_events`. **Surviving audit tables:** `entity_audit_logs`, `user_audit_logs`. Full list: advance_payments, annual_report_annex_data, annual_report_credit_points, annual_report_details, annual_report_expense_lines, annual_report_income_lines, annual_report_schedules, annual_report_status_history*, annual_reports, authority_contacts, binder_handover_binders, binder_handovers, binder_intake_edit_logs*, binder_intake_materials, binder_intakes, binder_lifecycle_logs*, binders, businesses, charges, client_records, correspondence_entries, deadline_rules, entity_audit_logs, entity_notes, idempotency_keys, invoices, legal_entities, notifications, password_reset_tokens, permanent_documents, person_legal_entity_links, persons, reminders, signature_audit_events*, signature_requests, tasks, tax_calendar_entries, user_audit_logs, users, vat_audit_logs*, vat_invoices, vat_work_items. (* = drop in cleanup)

**Cleanup downgrade recreation (Phase 9):** the down-migration must recreate each dropped table with its original columns, PK, FKs (to `binders`/`vat_work_items`/`annual_reports`/`signature_requests`/`users`, incl. the `vat_audit_logs.invoice_id → vat_invoices ON DELETE SET NULL`), and per-table indexes (each table indexes its parent FK + the timestamp column). Source of truth = `alembic/versions/3e2669e69e32_initial.py`.

## 9. Binding action matrix (Phase 0 deliverable)

`entity_type → action → old/new payload → metadata_json → actor_type → actor source`. Field diffs always in old/new; `client_record_id` mandatory where a client context exists. This is the binding spec for Phases 2–8 writers.

| Domain | entity_type | action(s) | old/new payload | metadata_json | actor_type | actor source |
|---|---|---|---|---|---|---|
| clients | client | created/updated/deleted/restored | changed client fields | client_record_id(self) | user | route current_user |
| businesses | business | created/updated/deleted/restored | business fields | client_record_id, business_id | user | current_user |
| legal_entities | legal_entity | updated | id_number/official_name/profile fields | client_record_id | user | owning client/business route |
| persons | person | updated | person fields | client_record_id (if linked) | user | route |
| person_legal_entity_links | person_legal_entity_link | created/deleted | role | client_record_id, legal_entity_id | user | route |
| authority_contacts | authority_contact | created/updated/deleted | contact fields | client_record_id, contact_id | user | route (**add actor to add/update**) |
| entity_notes | note | created/updated/deleted | note text | client_record_id, entity_type/entity_id | user | route |
| advance_payments | advance_payment | created/updated/amount_changed/marked_paid/deleted | expected/paid/override/status | client_record_id, period, tax_year | user | route (**add actor to create/update**) |
| charges | charge | issued/paid/canceled | status/amount | client_record_id, business_id | user | route |
| invoices | invoice | created | provider/external_id | client_record_id(via charge), invoice_id | user | route |
| vat_work_items | vat_work_item | created/status_changed/assigned/filed/amount_overridden | status/amounts/override | client_record_id, period, tax_year | user | created_by/performed_by |
| vat_invoices | vat_invoice | created/updated/amount_changed/deleted | net/vat/counterparty | client_record_id, vat_work_item_id, invoice_number, period | user | created_by |
| annual_reports | annual_report | created/updated/status_changed/submitted/deleted | status/fields | client_record_id, tax_year | user | created_by/changed_by |
| annual-report child rows | annual_report | income_line_added/updated/deleted; expense_line_added/updated/deleted; annex_line_updated/deleted; schedule_completed | line/schedule values | client_record_id, tax_year, section, line_id/schedule_id | user | parent route actor |
| binders | binder | created/marked_full/reopened/marked_ready_for_handover/reverted_ready/handed_over/deleted/restored | status field(s) | client_record_id, binder_id, field_name | user | actor_id/changed_by_user_id |
| binder_intakes | binder_intake | received/updated | intake fields | client_record_id, binder_id, field_name | user | received_by/changed_by |
| binder_handovers | binder_handover | created | recipient/period | client_record_id, binder_id(s) | user | created_by |
| documents | document | uploaded/replaced/approved/rejected/deleted | status/version | client_record_id, business_id, document_type, version | user | uploaded_by/approved_by/rejected_by (**add actor to delete**) |
| signature_requests | signature_request | created/sent/viewed/signed/declined/canceled/expired | status | per §8a forensic (client_record_id, signer_name, +ip/user_agent/content_hash per action) | user (advisor) / external_signer (signer) / system | advisor: current_user; signer: external; system: none |
| tasks | task | created/assigned/completed/canceled/deleted | status/assignment | client_record_id, source_domain, source_id | user / system | created_by_user_id / scheduler |
| correspondence | correspondence | created/updated/deleted | entry fields | client_record_id, business_id | user | created_by |
| notifications | notification | sent (user-initiated/significant only) | status | client_record_id | user / system | triggered_by / system |
| reminders | reminder | created/canceled/fired/failed | status | client_record_id (via source) | system (fired) / user (manual) | scheduler / route |
| tax_calendar | tax_calendar | generated (one per batch); manual edit = updated | — | firm-level / tax_year | system / user | materialization / route |
| deadline_rules (if UI-editable) | deadline_rule | updated | rule fields | firm-level | user | route |

## 10. AuditEntityRegistry preparation matrix (Phase 0 deliverable)

`entity_type → model/table → resolve_scope → allowed_roles → scope_policy → deleted_entity_policy → sensitive? → forensic visibility`. **Uses only the existing ADVISOR/SECRETARY role model + existing active/deleted client filtering. No owner/accountant permissions invented.**

| entity_type | model/table | resolve_scope | allowed_roles | scope_policy | deleted_entity_policy | sensitive? | forensic visibility |
|---|---|---|---|---|---|---|---|
| client | ClientRecord/client_records | self id → {client_id} | ADVISOR, SECRETARY | existing client-scope filter | soft: readable+flagged; hard: from audit metadata | no | n/a |
| business | Business/businesses | legal_entity→client | ADVISOR, SECRETARY | client-scope | soft/hard as above | no | n/a |
| legal_entity / person / person_legal_entity_link | resp. models | link→client (may be multi-client) | ADVISOR, SECRETARY | client-scope; multi-client union | soft/hard | no | n/a |
| authority_contact / note / correspondence | resp. | client_record_id column | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| advance_payment / charge / invoice | resp. | client_record_id (invoice via charge) | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| vat_work_item / vat_invoice | resp. | client_record_id (invoice via work_item) | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| annual_report (+children) | AnnualReport | client_record_id (+ children via parent) | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| binder / binder_intake / binder_handover | resp. | client_record_id (intake via binder) | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| document | PermanentDocument | client_record_id | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| signature_request | SignatureRequest | client_record_id | ADVISOR, SECRETARY | client-scope | soft/hard (hard via audit metadata) | **YES** | ip/user_agent/content_hash gated by §16 access_rule within current role model |
| task | Task | client_record_id | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| notification / reminder | resp. | client_record_id (reminder via source) | ADVISOR, SECRETARY | client-scope | soft/hard | no | n/a |
| tax_calendar / deadline_rule | resp. | firm-level (empty client_ids, firm_level=True) | ADVISOR, SECRETARY | firm-level (no client filter) | n/a | no | n/a |

Scope cases covered: **firm-level** (tax_calendar, deadline_rule), **one client** (most), **multi-client** (legal_entity/person via links), **soft-deleted** (live row with include_deleted), **hard-deleted** (`resolved_from="audit_metadata"`, client_ids from `metadata_json.client_record_id`).

> **Open item for Phase 2:** confirm whether SECRETARY has any audit-read restriction today. Current finding: both roles pass existing client-scoped auth on these surfaces; no per-role audit restriction exists in code beyond route auth + client filtering. The registry encodes exactly that — it does not add new restrictions.

## 11. Signature audit replacement inventory

- **Writers (9 `append_audit_event`):** `signature_request_creation_service.py:114,122` (created/sent), `signature_request_signer_service.py:32,53,88,126` (viewed/signed/declined etc.), `signature_request_admin_service.py:45,68` (canceled/expired/resend), `signature_request_service.py:79`.
- **Readers:** `TimelineRepository.list_signature_lifecycle_events` (`timeline_repository.py:43-58`, joins SignatureRequest×SignatureAuditEvent, filters `_SIGNATURE_LIFECYCLE_TYPES`) → `timeline_client_aggregator.py:32` → `signature_request_lifecycle_event` builder (`timeline_client_builders.py:87`); **embedded** `audit_trail` via `signature_request_service.get_audit_trail` → `repo.list_audit_events` → `build_with_audit` (`signature_request_response_builder.py:41-45`) on `GET /signature-requests/{id}`.
- **Frontend:** `SignatureRequestAuditDrawer.tsx` consumes embedded `audit_trail` (`generated.ts:7135` `audit_trail: SignatureAuditEventResponse[]`).
- **Replacement response shape (Phase 6):** keep `SignatureRequestWithAuditResponse`; new embedded item = `{ action, actor_type, actor_display_name, performed_at, note, ...forensic from metadata_json }`. Items fetched via `AuditTrailService` and passed into `build_with_audit` (which already accepts items — no repo dependency).
- **Forensic metadata mapping (per §8a):** `event_type → signature_request.<event>`; `actor_type advisor→user / signer→external_signer / system→system`; move `actor_name → actor_display_name`, and `ip_address/user_agent/content_hash/signed_document_key → metadata_json` per action (ip/user_agent on viewed/signed/declined when available; content_hash on signed, else `content_hash_missing=true`).
- **Seed:** `app/seed/builders/demo/signature_requests.py:229` writes `SignatureAuditEvent` → repoint to EntityAuditLog.

## 12. Timeline / dashboard inventory

**Timeline sources today (`app/timeline/`):** client created (live), business changed (EntityAuditLog), charge created/issued/paid + invoice (live), annual_report status (AnnualReportStatusHistory via `timeline_repository.list_annual_report_status_events:63`), binder received/handed_over (live), binder lifecycle (BinderLifecycleLog), signature lifecycle (SignatureAuditEvent via `list_signature_lifecycle_events:43`), document uploaded (live PermanentDocument), notifications (live). Dedup: `_DEDUP_ACTIONS` in `timeline_audit_aggregator.py`.

**Categories moving to EntityAuditLog:** annual_report status (Phase 4), binder lifecycle (Phase 5), signature lifecycle (Phase 6). VatAuditLog + BinderIntakeEditLog are **not** read by timeline.

**Dashboard (`dashboard_recent_activity_service.py`):** reads EntityAuditLog (`list_recent`) **+** BinderLifecycleLog (`binder_lifecycle_log_repo.list_recent`, line 95), merges top 5; special `_serialize_binder` (line 166), `_binder_label` (179), dual-timestamp `_timestamp` (212). After Phase 5: EntityAuditLog-only, drop `_serialize_binder`/binder repo/dual timestamp.

**Event-source registry gaps / duplicate risks:** binder `handed_over` exists both as a live builder and (post-merge) as `binder.handed_over` audit → registry must assign it to **one** source (live builder); `_DEDUP_ACTIONS` removed only after single-source tests prove zero duplicates (Phase 7).

**Required tests:** single-source-per-category (no duplicates), timeline status/binder/signature events present after re-point, dashboard EntityAuditLog-only.

## 13. Actor availability inventory

| Domain | Function | Has actor? | Param | actor_type | Fix |
|---|---|---|---|---|---|
| advance_payments | `create_payment_for_client:117` | **NO** | — | user | **add `actor_id`** |
| advance_payments | `update_payment_for_client:193` (`**fields`) | **NO** | — | user | **add `actor_id`** |
| advance_payments | `delete_payment_for_client:238` | yes | `actor_id` | user | reuse |
| authority_contacts | `add_contact:19` | **NO** | — | user | **add `actor_id`** |
| authority_contacts | `update_contact:45` | **NO** | — | user | **add `actor_id`** |
| authority_contacts | `delete_contact:80` | yes | `actor_id` | user | reuse |
| permanent_documents | `upload_document:95` | yes | `uploaded_by` | user | reuse (wire to writer) |
| permanent_documents | `replace_document:309` / approve / reject | yes | actor fields | user | reuse |
| permanent_documents | `delete_document:302` | **NO** | — | user | **add `actor_id`** |
| legal_entities / persons / links | (mutated via client/business services) | partial | — | user | thread `actor_id` from owning route |
| vat_work_items / vat_invoices | vat services | yes | `created_by`/`performed_by` | user | reuse |
| annual_reports (+children) | AR services | yes | `created_by`/`changed_by`/`actor_id` | user | reuse |
| binders / intakes | binder services | yes | `actor_id`/`changed_by_user_id`/`received_by` | user | reuse |
| charges / invoices | billing/invoice services | yes | `created_by`/`issued_by`/`paid_by`/`canceled_by` | user | reuse |
| correspondence / notes / tasks | resp. services | yes | `created_by`/`actor_id`/`created_by_user_id` | user | reuse |
| notifications | notification service | yes | `triggered_by` | user/system | reuse; system→None |
| reminders / tax_calendar | scheduling/materialization | no | — | system | actor_type=system, performed_by=None |
| signature (external) | signer service | n/a (external) | — | external_signer | performed_by=None, identity in metadata + actor_display_name |

**Missing-actor fixes confirmed (5 surfaces):** advance_payment create + update, authority_contact add + update, permanent_document delete. All have a route `current_user` available to supply `actor_id` + `actor_display_name`.

## 14. Blockers

**None.** No baseline failure; no inventory contradiction with the plan. The two `unused (4)` / `schema (43)` script findings are pre-existing, unrelated, informational.

## 15. Risks (carried into implementation)

1. **Signature forensic/legal parity** — embedded drawer + timeline both depend on signature audit; per-action metadata mapping must preserve ip/user_agent/content_hash (§8a).
2. **Timeline duplicate events** — `handed_over` and similar exist as both live + audit; single-source registry + tests required before removing `_DEDUP_ACTIONS`.
3. **JSON serializer/reader switch** — must land in the same step as the JSONB column change, with portable `JSON().with_variant(JSONB)` model type to keep SQLite `create_all` working.
4. **Phase-1 NOT NULL sequencing** — enforce `actor_type NOT NULL` only after all writers populate it (migrations 1a→1b).
5. **Downgrade** — fail-safe assert on `performed_by IS NULL` before restoring NOT NULL.
6. **`annualreportstatus` shared enum** — must not be dropped in cleanup.
7. **Layering** — keep registry/scope/authorization/redaction in `AuditTrailService`; repositories DB-only; signature builder fed items via the service.
8. **Frontend per-phase migration** — deleted routes (VAT/binder/AR) must switch their frontend consumer in-phase; no temporary legacy routes.

## 16. Recommendation

**Phase 1 is SAFE TO START.** Baseline is green, inventory is complete and consistent with the plan, the enum landscape is simpler than assumed (zero enum drops), and the serializer/reader + migration sequencing risks are understood and already encoded in the plan (§1b). Proceed to Phase 1 (schema add/alter only, migrations 1a→1b, serializer/reader changes, writer actor support) — **no legacy table drops in Phase 1.**
