# Audit Refactor — Implementation Plan

Status: **planning only**. No code, migration, model, schema, seed, OpenAPI, or frontend change is made by this document.

DB is dev/seed-only → **no backfill, no dual-write, no backward compatibility, no legacy aliases**. Target end state: exactly **two** audit models.

- **EntityAuditLog** — every business mutation **plus selected evidence events** (state-machine / legal-evidence transitions: `signature_request.viewed/signed/declined`, `annual_report.submitted`, `charge.paid`, `binder.handed_over`, …). Business mutation = a field/state change on a domain entity; evidence event = a point-in-time fact worth proving even when no field "diff" is interesting on its own.
- **UserAuditLog** — auth/security/admin-access only (login, logout, password reset, user create/activate/deactivate/role-change).

`VatAuditLog`, `AnnualReportStatusHistory`, `BinderLifecycleLog`, `BinderIntakeEditLog`, `SignatureAuditEvent` are deleted and merged into `EntityAuditLog`.

### Locked decisions
- **Read routes — collapse to generic.** Delete per-domain audit read routes; expand the single `GET /audit/{entity_type}/{entity_id}` to cover all entity types, one response schema.
- **Action granularity — rich semantic.** Specific verbs (`vat_invoice.amount_changed`, `advance_payment.marked_paid`, `signature_request.signed`) + `domain.created/updated/deleted` for plain edits. Field diffs always in `old_value`/`new_value`.
- **Migration — additive, not a rewrite.** The repo has a single initial migration (`alembic/versions/3e2669e69e32_initial.py`). It is **not** rewritten. A **new** migration alters `entity_audit_logs` and drops the five old audit tables.

---

## Target models

**EntityAuditLog** (`app/audit/models/audit_entity_audit_log.py`, table `entity_audit_logs`):
```
id, entity_type:str, entity_id:int,
action:str,                       # rich semantic "domain.action"
performed_by:int|None (FK users.id, nullable),
actor_type:str,                   # user | system | external_signer
old_value:JSONB|None, new_value:JSONB|None, metadata_json:JSONB|None,
note:str|None, performed_at:datetime
```
Delta vs today: `old_value`/`new_value` Text→JSONB; **add** `actor_type`, `metadata_json`; `performed_by` → Integer FK `users.id`, **nullable** (was non-null). Indexes: `(entity_type, entity_id, performed_at)`, `(action, performed_at)`, `(performed_by, performed_at)`, `(performed_at)`.

**UserAuditLog** (`app/users/models/user_audit_log.py`, table `user_audit_logs`): keep. Only change: `metadata_json` Text→JSONB. Keep `action`/`status` as the existing closed enums (`AuditAction`/`AuditStatus`). Existing columns suffice.

**Decision — user activate/deactivate stays UserAuditLog-only.** User create/activate/deactivate/role-change are admin/security-access events and remain in `UserAuditLog` exclusively. **No dual-write** to `EntityAuditLog`. There is **no `ENTITY_USER` constant** in the final entity_type set (§6). Auditing user-entity changes in `EntityAuditLog` is explicitly out of scope; it would be a future, separate decision tied to a genuine business-facing user-profile screen.

---

## 1. Audit models inventory (exact)

7 models found (target 2). Verified by class scan.

| Model | File | Table | Writers (sites) | Readers | API | Tests | Target |
|---|---|---|---|---|---|---|---|
| **EntityAuditLog** | `app/audit/models/audit_entity_audit_log.py` | `entity_audit_logs` | `EntityAuditWriter` (14 `record_*` sites) | `AuditTrailService`, timeline aggregator, dashboard recent-activity | `GET /audit/{type}/{id}` | `tests/audit/*` (3) | **KEEP + upgrade** |
| **UserAuditLog** | `app/users/models/user_audit_log.py` | `user_audit_logs` | `AuditLogService.log` (10 sites) | `AuditLogService.list_logs` | `GET /users/audit-logs` | `tests/users/...audit*` (2) | **KEEP** (metadata_json→JSONB) |
| **VatAuditLog** | `app/vat/models/vat_audit_log.py` | `vat_audit_logs` | `work_item_repo.append_audit` (12 sites) | `VatReportService.get_audit_trail_enriched` | `GET /vat/work-items/{id}/audit` | `tests/vat/api/test_vat_reports_audit.py` | **DELETE → EntityAuditLog** |
| **AnnualReportStatusHistory** | `app/annual_reports/models/annual_report_status_history.py` | `annual_report_status_history` | `append_status_audit_entry` (4 sites) | `AnnualReportService.get_report_audit`, timeline | `GET /annual-reports/{id}/audit` | `tests/annual_reports/api/test_annual_report_audit.py` | **DELETE → EntityAuditLog** |
| **BinderLifecycleLog** | `app/binders/models/binder_lifecycle_log.py` | `binder_lifecycle_logs` | `BinderLifecycleService._append_log` (9 sites) | `BinderAuditService`, timeline, dashboard | `GET /binders/{id}/audit` | `tests/binders/api/test_binder_audit.py`, `tests/binders/service/test_binder_lifecycle_*` | **DELETE → EntityAuditLog** |
| **BinderIntakeEditLog** | `app/binders/models/binder_intake_edit_log.py` (+ `binder_intake_edit_log_repository.py`, `binder_intake_edit_service.py`) | `binder_intake_edit_logs` | `edit_intake` (2 sites) | none (no read route) | written via `PATCH /binders/{id}/intakes/{id}` | none direct | **DELETE → EntityAuditLog** |
| **SignatureAuditEvent** | `app/signature_requests/models/signature_request.py` | `signature_audit_events` | `SignatureRequestAuditMixin.append_audit_event` (10 sites) | `TimelineRepository.list_signature_lifecycle_events` | timeline only (no dedicated audit route) | `tests/timeline/service/test_timeline_signature_lifecycle.py` | **DELETE → EntityAuditLog** |

Repos/services kept/extended: `EntityAuditLogRepository`, `EntityAuditWriter`, `AuditTrailService`, `UserAuditLogRepository`, `AuditLogService`.
Repos/services/schemas deleted: `VatAuditLogRepository` + `vat/schemas/vat_audit.py`; `AnnualReportStatusAuditRepository` + `AnnualReportAuditEntry`/`AnnualReportAuditListResponse`; `BinderLifecycleLogRepository` + `BinderAuditService` + `BinderAuditEntry`/`BinderAuditResponse`; `BinderIntakeEditLogRepository` + `BinderIntakeEditService` write path; `SignatureRequestAuditMixin` + `SignatureAuditEventResponse`/`SignatureRequestWithAuditResponse`.

## 2. Call-site map (exact counts)

Write call-sites (verified by grep, excluding tests):

| Source | Sites |
|---|---|
| `EntityAuditWriter.record_*` (EntityAuditLog today) | 14 |
| `work_item_repo.append_audit` → VatAuditLog | 12 |
| `BinderLifecycleService._append_log` → BinderLifecycleLog | 9 |
| `append_audit_event` → SignatureAuditEvent | 10 |
| `append_status_audit_entry` → AnnualReportStatusHistory | 4 |
| `edit_intake` → BinderIntakeEditLog | 2 |
| **Business-target write sites (→ EntityAuditLog)** | **51** |
| `AuditLogService.log` → UserAuditLog (auth/admin) | 10 |
| **Total audit write sites** | **61** |

Read/aggregation call-sites: `AuditTrailService.get_entity_audit_trail`, `VatReportService.get_audit_trail_enriched`, `BinderAuditService.get_binder_audit`, `AnnualReportService.get_report_audit`, `AuditLogService.list_logs`, timeline aggregator (`timeline_audit_aggregator.build_entity_audit_events` + `TimelineRepository` lifecycle/status/signature readers), dashboard `RecentActivityService.build`. = **8 distinct read paths.**

API call-sites: **8** (see §3). Seed call-sites: **3** — `app/seed/builders/demo/vat.py`, `app/seed/builders/demo/binders.py` (lifecycle + intake-edit), `app/seed/orchestrator.py`. Test files in audit/timeline/dashboard/lifecycle area: **43** (exact, from `find tests`). The count that *directly assert audit/history behavior and require changes* is **to be finalized in Phase 0** (file-by-file triage); §13 lists the currently-identified set without claiming completeness.

## 3. Audit/history/timeline routes (8)

| # | Route | Method | File | Source today | After |
|---|---|---|---|---|---|
| 1 | `/audit/{entity_type}/{entity_id}` | GET | `app/audit/api/audit_routes.py:18` | EntityAuditLog | **STAYS + EXPANDS** — `ALLOWED_READ_ENTITY_TYPES` grows to all types; sole entity-audit read route |
| 2 | `/users/audit-logs` | GET | `app/users/api/user_routes_audit.py:24` | UserAuditLog | **STAYS** unchanged |
| 3 | `/vat/work-items/{id}/audit` | GET | `app/vat/api/vat_routes_queries.py:184` | VatAuditLog | **DELETE** → `/audit/vat_work_item/{id}` |
| 4 | `/binders/{id}/audit` | GET | `app/binders/api/binder_routes_audit.py:26` | BinderLifecycleLog | **DELETE** → `/audit/binder/{id}` |
| 5 | `/annual-reports/{id}/audit` | GET | `app/annual_reports/api/annual_report_routes_status.py:129` | AnnualReportStatusHistory | **DELETE** → `/audit/annual_report/{id}` |
| 6 | `/binders/{id}/intakes` | GET | `app/binders/api/binder_routes_audit.py:56` | BinderIntake (not edit log) | **STAYS** (reads intake rows) |
| 7 | `/clients/{id}/timeline` | GET | `app/timeline/api/timeline_routes.py:19` | 7 sources | **STAYS** as product timeline; sources re-pointed (§4) |
| 8 | `/dashboard/overview` (`recent_activity`) | GET | `app/dashboard/api/dashboard_routes_overview.py` | EntityAuditLog + BinderLifecycleLog | **CHANGES** — EntityAuditLog only (§5) |

Also: `PATCH /binders/{id}/intakes/{id}` (`binder_routes_audit.py`) currently **writes** BinderIntakeEditLog → repointed to EntityAuditLog write (no read route affected).

Response compatibility required: **no**. Routes 3–5 + their schemas are deleted.

## 4. Timeline — redesign with one source of truth per category (correction #5)

`app/timeline/services/timeline_service.py` aggregates the event categories listed in the registry below (per client) and is a **product timeline**, not a raw audit viewer. Files: `timeline_service.py`, `timeline_audit_aggregator.py` (holds `_DEDUP_ACTIONS`), `timeline_client_builders.py` (has `entity_audit_changed_event`), `timeline/repositories/timeline_repository.py`, `timeline_labels.py`. Client context anchor = `client_record_id` input; per-source via direct column or parent FK.

Today the same logical event can arrive from **both** a live-table builder and an EntityAuditLog row, patched up by `_DEDUP_ACTIONS`. The refactor **replaces dedup-by-patch with an explicit event-source registry**: each event category has exactly **one** source. Audit rows whose category is owned by a live builder are simply not queried for timeline (they remain in the audit-trail view).

**Timeline event-source registry (authoritative — one source each):**

| Event category | Single source | Note |
|---|---|---|
| client created | live builder (`client_created_event`) | richer than `client.created` audit |
| business changed | EntityAuditLog `business.*` | no live builder |
| charge created/issued/paid + invoice attached | live builders (`charge_*`, `invoice_attached`) | `charge.*` audit NOT queried for timeline |
| annual_report status changed / submitted | EntityAuditLog `annual_report.status_changed`/`submitted` | replaces AnnualReportStatusHistory |
| binder received / handed_over | live builders | from `Binder`/`BinderHandover` |
| binder lifecycle (marked_full/reopened/ready/reverted) | EntityAuditLog `binder.*` | replaces BinderLifecycleLog; **excludes** received/handed_over (owned above) |
| signature lifecycle (sent/viewed/signed/declined/canceled/expired) | EntityAuditLog `signature_request.*` | replaces SignatureAuditEvent |
| document uploaded | live `PermanentDocument` | |
| notifications sent/failed | live `Notification` | |

**Recommendation — keep aggregator + builders; re-point 3 broken streams (annual-report status, binder lifecycle, signature) to EntityAuditLog; then retire `_DEDUP_ACTIONS`.** `_DEDUP_ACTIONS` is removed **only after** the event-source registry above covers every category and timeline tests prove zero duplicate events (see Phase 7) — not in the same step as the re-point. VatAuditLog + BinderIntakeEditLog were never read by timeline → no timeline impact.

- **Child-entity client context:** resolved via **`metadata_json.client_record_id`** written at audit time (§8) — timeline needs no per-entity joins. New repo query `list_for_client_context(client_record_id, entity_types, business_ids)` reading `metadata_json->>'client_record_id'`.
- Files changed: `timeline_service.py`, `timeline_audit_aggregator.py`, `timeline_client_builders.py`, `timeline_repository.py`, `timeline_labels.py`. Files deleted: none in timeline (deletions are old models/repos).
- Risk: Hebrew label parity for the re-pointed semantic actions; the registry must be enforced so a live builder and an audit row never both fire for one category.

## 5. Dashboard recent activity — redesign

`app/dashboard/services/dashboard_recent_activity_service.py` today fetches 20 recent EntityAuditLog + 20 recent BinderLifecycleLog (imports at lines 24/25/27/28), merges, top 5; special `_serialize_binder` branch (negated ids, `field_name=="location_status"` detection, separate client lookup, `performed_at or changed_at` glue).

**Target:** EntityAuditLog only via `list_recent(limit)`. Remove `_serialize_binder`, the BinderLifecycleLog import/repo, negated-id hack, dual timestamp glue. Binder activity now arrives as `entity_type="binder"` + semantic `action` → label via `_ACTION_LABELS`, href + `client_name` via `metadata_json` (`binder_id`, `client_record_id`). Single serialization path. Files: `dashboard_recent_activity_service.py`. Risk: `_ACTION_LABELS`/href maps must gain binder semantic actions; all recent-activity-eligible writes must carry `metadata_json.client_record_id`.

## 6. ENTITY_* constants — final

Canonical: `app/audit/audit_constants.py`. Rule: own `entity_type` only with an independent screen/route; child rows → parent entity_type + `metadata_json.<child>_id`.

| Table/domain | entity_type | Granularity | Reason |
|---|---|---|---|
| client_records | `client` (exists) | row | own screen |
| businesses | `business` (exists) | row | own screen |
| legal_entities | `legal_entity` | row | identity/tax-profile data |
| persons | `person` | row | identity data |
| person_legal_entity_links | `person_legal_entity_link` | row | ownership/role changes |
| authority_contacts | `authority_contact` | row | own CRUD |
| entity_notes | `note` | row | own CRUD |
| advance_payments | `advance_payment` | row | own detail page |
| charges | `charge` (exists) | row | own screen |
| **invoices** | **`invoice`** | row | **resolved (correction #3): `app/invoices/api/invoice_routes.py` exposes independent POST+GET API → own entity_type. Minimal lifecycle (`invoice.created`); if that API is ever removed, fold to `charge` + `metadata_json.invoice_id`.** |
| vat_work_items | `vat_work_item` | row | own screen |
| vat_invoices | `vat_invoice` | row | own edit modal |
| annual_reports | `annual_report` (exists) | row | own screen |
| annual_report_details/income/expense/credit_points/schedules | `annual_report` | **aggregate** | child; `metadata_json.section` + line/schedule id |
| annual_report_annex_data | `annual_report` | aggregate | bulk save → one `annual_report.updated`, `section="annex_data"` |
| binders | `binder` | row | own screen |
| binder_intakes | `binder_intake` | row | own intake flow |
| binder_intake_materials | `binder_intake` | aggregate | child of intake |
| binder_handovers | `binder_handover` | row | handover event |
| binder_handover_binders | — | — | junction; audit at handover |
| permanent_documents | `document` | row | upload/approve/reject lifecycle |
| signature_requests | `signature_request` | row | lifecycle + forensic |
| tasks | `task` | row | own screen |
| correspondence_entries | `correspondence` | row | own CRUD |
| notifications | `notification` | row (partial) | user-initiated / business-significant only |
| reminders | `reminder` | row (partial) | created/canceled/fired-failed only |
| tax_calendar_entries | `tax_calendar` | aggregate | one `tax_calendar.generated`/batch; manual edit = row |
| deadline_rules | `deadline_rule` (conditional) | row | only if UI-editable |
| users | — (no `ENTITY_USER`) | — | auth/admin → UserAuditLog only |
| idempotency_keys / password_reset_tokens | — | none | infra |

Final `ENTITY_*` set = spec §9 list + `invoice` + `deadline_rule` (conditional). **No `ENTITY_USER`** (user events stay UserAuditLog-only). `ALLOWED_READ_ENTITY_TYPES` expands to the full set (gates the generic read route).

## 7. ACTION_* constants — final (rich semantic)

Convention `domain.action`, constants only, **no raw strings in services**. Plain field edits → `domain.updated` (diffs in old/new); lifecycle/evidence → specific verbs.

| Action(s) | Replaces |
|---|---|
| `client.created/updated/deleted/restored` | existing CREATED/UPDATED/DELETED/RESTORED (client) |
| `business.created/updated/deleted/restored` | existing |
| `legal_entity.updated`, `person.updated`, `person_legal_entity_link.created/deleted` | new |
| `advance_payment.created/updated/amount_changed/marked_paid/deleted` | new |
| `charge.issued/paid/canceled` | existing ISSUED/PAID/CANCELED |
| `invoice.created` | new |
| `vat_work_item.created/status_changed/assigned/filed/amount_overridden` | VatAuditLog actions |
| `vat_invoice.created/updated/amount_changed/deleted` | VatAuditLog actions |
| `annual_report.created/updated/status_changed/submitted/deleted` | STATUS_CHANGED + AnnualReportStatusHistory |
| `annual_report.updated` + `metadata_json.section` | DETAIL_UPDATED, INCOME_*, EXPENSE_*, ANNEX_LINE_* |
| `document.uploaded/replaced/approved/rejected/deleted` | new |
| `signature_request.created/sent/viewed/signed/declined/canceled/expired` | SignatureAuditEvent `event_type` (evidence events) |
| `binder.created/marked_full/reopened/marked_ready_for_handover/reverted_ready/handed_over/deleted/restored` | BinderLifecycleLog field rows |
| `binder_intake.received/updated` | BinderIntakeEditLog |
| `task.created/assigned/completed/canceled/deleted` | new |
| `correspondence.created/updated/deleted` | new |
| `notification.sent`, `reminder.created/canceled/failed`, `tax_calendar.generated` | new (partial) |

Annual-report financial-line + annex constants (`ACTION_INCOME_*`, `ACTION_EXPENSE_*`, `ACTION_ANNEX_LINE_*`) collapse into `annual_report.updated` + `metadata_json.section`/line id. Keep transition verbs (`status_changed`, `submitted`) distinct.

## 8. metadata_json contract

`client_record_id` is **mandatory whenever a client context exists** (timeline/dashboard depend on it). The changed field itself goes in `old_value`/`new_value`, never metadata.

| entity_type | required | optional | used by | risk if missing |
|---|---|---|---|---|
| vat_invoice | client_record_id, vat_work_item_id, invoice_number, period, tax_year | business_id, source | timeline, audit view | can't place under client |
| vat_work_item | client_record_id, period, tax_year | source | timeline, dashboard | same |
| advance_payment | client_record_id, period, tax_year | annual_report_id, source | timeline | same |
| annual_report (+children) | client_record_id, tax_year | section, line_number, schedule | timeline, audit view | child orphaned |
| document | client_record_id | business_id, annual_report_id, binder_id, document_type, version | timeline | href/label gaps |
| signature_request | client_record_id, signer_name, signer_email (always) | per-action forensic (see below) | timeline, **forensic/legal** | evidence loss |
| binder / binder_intake | client_record_id, binder_id | binder_number, period_start/end, field_name | timeline, dashboard | dashboard href breaks |
| task | client_record_id | source_domain, source_id, assigned_to_user_id, assigned_role | timeline | context loss |
| charge / invoice | client_record_id | business_id, annual_report_id, invoice_id | dashboard, timeline | href breaks |

**Signature forensic metadata is per-action (not blanket):**

| signature_request action | required metadata | notes |
|---|---|---|
| created, sent | client_record_id, signer_name, signer_email, business_id?, annual_report_id?, document_id? | no ip/user_agent (advisor-initiated) |
| viewed, signed, declined | + ip_address, user_agent **when available** | signer-initiated; capture client forensics if the request carried them |
| signed | + content_hash (**strongly required**), signed_document_key | content_hash proves what was signed; treat a missing hash on `signed` as a defect |
| canceled, expired | client_record_id (+ reason for canceled) | no ip/user_agent required |

## 9. actor / user_id availability

Approach = **explicit service-level writes** (no contextvars/after_flush/auto-audit). Audit needs `performed_by` (nullable) + `actor_type` (`user`/`system`/`external_signer`); actor flows from route `current_user.id`.

| Domain | File | Actor today | Action |
|---|---|---|---|
| legal_entities / persons / links | mutated via client/business services | partial | thread `actor_id` from owning route |
| advance_payments | `advance_payments/services/advance_payment_service.py` | delete has `actor_id`; create/update **no** | **add `actor_id`** to create/update |
| permanent_documents | `documents/permanent_documents/services/permanent_document_service.py` | `uploaded_by/approved_by/rejected_by` on model, not passed to a writer | pass actor into audit write |
| vat_work_items / vat_invoices | `vat/services/*` | `created_by`/`performed_by` present | reuse as `performed_by` |
| annual_reports (+children) | `annual_reports/services/*` | `created_by`/`actor_id`/`changed_by` present | reuse |
| binders / binder_intakes | `binders/services/binder_lifecycle_service.py`, `binder_intake_service.py` | `actor_id`/`changed_by_user_id`/`received_by` present | reuse |
| charges / invoices | `charges/services/charge_billing_service.py`, `invoices/services/invoice_service.py` | `created_by/issued_by/paid_by/canceled_by` | reuse |
| authority_contacts | `authority_contacts/services/authority_contact_service.py` | delete has `actor_id`; add/update **no** | **add `actor_id`** |
| correspondence | `communications/services/correspondence_service.py` | `created_by` present | reuse |
| notes | `notes/services/note_entity_note_service.py` | `created_by`/`actor_id` present | reuse |
| tasks | `tasks/services/task_service.py` | `created_by_user_id` present | reuse |
| notifications | `notifications/services/notification_service.py` | `triggered_by` present | reuse; auto → `actor_type="system"`, `performed_by=None` |
| reminders / tax_calendar | scheduling / materialization | system | `actor_type="system"`; manual edit → route actor |
| signature (external) | `signature_requests/*` | external signer, no user | `actor_type="external_signer"`, `performed_by=None`, identity in metadata |

Missing-actor services to fix: **advance_payment create/update, authority_contact add/update, permanent_document writer, legal_entity/person/link mutations** → add `actor_id: int` param fed by `current_user.id`.

## 10. Tables needing EntityAuditLog (from `3e2669e69e32_initial.py`, 53 tables)

**Critical:** legal_entities, persons, person_legal_entity_links, advance_payments, vat_work_items, vat_invoices, permanent_documents, annual_reports + child tables (aggregate at parent), charges, invoices, signature_requests.
**High:** client_records, businesses, authority_contacts, entity_notes.
**Medium:** binders, binder_intakes, binder_intake_materials (aggregate), binder_handovers, tasks, correspondence_entries.
**Low/conditional:** notifications (user-initiated only), reminders (significant events only), tax_calendar_entries (batch = one event), deadline_rules (only if UI-editable).
**No audit:** idempotency_keys, password_reset_tokens, binder_handover_binders (junction), users (→ UserAuditLog), tax_calendar materialized rows per-batch.

Aggregate reminders: annex bulk save → one `annual_report.updated`; tax_calendar generation → one `tax_calendar.generated`; notification internal retry → none.

## 11. Old-model deletion checks

| Old model | Typed columns read by API/timeline | EntityAuditLog preserves? | Actions | Metadata | Remove | Risk |
|---|---|---|---|---|---|---|
| VatAuditLog | work_item_id, action, old/new, note, invoice_id, performed_by/at | **Yes** | `vat_work_item.*`, `vat_invoice.*` | client_record_id, vat_work_item_id, invoice_number, period, tax_year | route, `vat_audit.py` schema, `VatAuditLogRepository`, seed `create_vat_audit_logs`; rewrite test | low |
| AnnualReportStatusHistory | from_status, to_status, changed_by, note, occurred_at | **Yes** | `annual_report.status_changed` (old/new=status) | client_record_id, tax_year, form_type | route, schema, `AnnualReportStatusAuditRepository`; rewrite test | low |
| BinderLifecycleLog | field_name, old/new, changed_by, changed_at, notes | **Yes** (semantic actions + `metadata.field_name`) | `binder.marked_full/...` | client_record_id, binder_id, field_name | `/binders/{id}/audit`, `BinderAuditResponse`/`BinderAuditEntry`, repo, `BinderAuditService`, dashboard branch; rewrite test | medium (label/registry parity) |
| BinderIntakeEditLog | field_name, old/new, changed_by | **Yes** | `binder_intake.updated` | client_record_id, binder_id, field_name | model, repo, `BinderIntakeEditService` write path | low (no read route) |
| **SignatureAuditEvent** | event_type, actor_type, actor_id, actor_name, ip_address, user_agent, notes, occurred_at | **Yes — verify first** | `signature_request.*` (evidence) | per-action (§8): always client_record_id/signer_name/signer_email; ip/user_agent for viewed/signed/declined when available; content_hash strongly required for signed; signed_document_key on signed | model, mixin, `SignatureAuditEventResponse`/`SignatureRequestWithAuditResponse`; update `TimelineRepository.list_signature_lifecycle_events`; rewrite test | **high — legal/forensic**: `actor_type` maps `advisor→user`, `signer→external_signer`, `system→system`; ip/user_agent/actor_name MUST move to metadata_json (per-action); **confirm no frontend signature-audit-trail view depends on typed columns before deleting** |

## 12. New repositories/services shape

`EntityAuditLogRepository` (`app/audit/repositories/audit_entity_audit_log_repository.py`): keep `append`, `get_audit_trail`, `count_audit_trail`, `list_all_by_entities`, `list_recent`. **Add** `list_by_entity`, `list_by_entities`, `list_for_client_context(client_record_id, entity_types=None, business_ids=None)` (reads `metadata_json->>'client_record_id'`), `list_recent_activity(limit)`. `append` gains `actor_type`, `metadata_json`; old/new accept dict (JSONB).

`EntityAuditWriter` (`app/audit/services/audit_entity_audit_writer_service.py`): keep `record_create/update/delete/restore/status_change`; **add** `record_action(entity_type, entity_id, *, action, actor_id, actor_type="user", old_value, new_value, metadata_json, note)` and `record_external_action(...)` (actor_type="external_signer", performed_by=None). All helpers gain `metadata_json` + `actor_type` (default `user`).

Delete: `VatAuditLogRepository`, `AnnualReportStatusAuditRepository`, `BinderLifecycleLogRepository`, `BinderIntakeEditLogRepository`, `BinderAuditService`, `SignatureRequestAuditMixin`, `BinderIntakeEditService` write path + their schemas.

Canonical names kept (do **not** rename despite spec's example paths): `app/audit/models/audit_entity_audit_log.py`, `.../repositories/audit_entity_audit_log_repository.py`, `.../services/audit_entity_audit_writer_service.py`, `.../audit_constants.py`, `.../services/audit_trail_service.py`.

## 13. Tests

Audit-coupled files needing updates (subset of the 43 in audit/timeline/dashboard/lifecycle areas):

| File | Breaks? | Update |
|---|---|---|
| `tests/audit/test_entity_audit_writer.py` | yes | JSONB metadata, actor_type, external_signer, nullable performed_by |
| `tests/audit/test_audit_endpoint.py`, `test_audit_endpoint_filters.py` | yes | expanded entity types, new fields |
| `tests/core/test_openapi_audit_paths.py` | yes | deleted routes drop from spec, generic route expands |
| `tests/vat/api/test_vat_reports_audit.py` | yes (route deleted) | rewrite → `/audit/vat_work_item/{id}` |
| `tests/annual_reports/api/test_annual_report_audit.py` | yes (deleted) | rewrite → generic |
| `tests/annual_reports/service/test_annual_report_generic_audit.py`, `test_financial_audit_snapshots.py` | yes | `annual_report.updated` + section |
| `tests/binders/api/test_binder_audit.py` | yes (deleted) | rewrite → generic |
| `tests/binders/service/test_binder_lifecycle_service.py`, `test_binder_lifecycle_notification.py` | yes | semantic `binder.*` actions |
| `tests/timeline/service/test_timeline_signature_lifecycle.py` | yes | re-source from EntityAuditLog |
| `tests/timeline/*` (repository, event_builders, get_client_timeline) | yes | event-source registry, drop dedup |
| `tests/dashboard/service/test_recent_activity_service.py`, `tests/dashboard/api/test_dashboard_extended.py` | yes | EntityAuditLog-only path |
| `tests/charges/service/test_billing_audit.py`, `tests/charges/api/test_charges_api_lifecycle.py` | minor | action namespacing |
| `tests/users/api/test_user_audit_logs.py`, `tests/users/services/test_audit_log_service.py` | minor | metadata_json JSONB |

**New tests:** EntityAuditLog JSONB metadata round-trip; nullable `performed_by` + actor_type system/external_signer; VAT write → EntityAuditLog; annual-report status → EntityAuditLog; signature signed → EntityAuditLog w/ forensic metadata; binder lifecycle → EntityAuditLog; dashboard recent-activity reads EntityAuditLog only; timeline reads each category from its single registry source; UserAuditLog stays auth/admin-only.

## 14. OpenAPI / frontend impact (plan only — do not regen)

| Route | Old → New | Frontend type | Regen? |
|---|---|---|---|
| `/audit/{type}/{id}` | + actor_type, metadata_json, JSONB old/new | audit hook types | **yes** |
| `/vat/work-items/{id}/audit` | **deleted** | VAT audit hook → generic | **yes** |
| `/binders/{id}/audit` | **deleted** | binder audit hook → generic | **yes** |
| `/annual-reports/{id}/audit` | **deleted** | report audit hook → generic | **yes** |
| `/dashboard/overview` (`recent_activity`) | source change; item shape intended unchanged | dashboard hook | **verify** (correction #6): confirm `activity_type` enum/label set unchanged after binder semantic actions added; regen if the recent_activity item schema shifts |
| `/clients/{id}/timeline` | sources change; `event_type` set may gain/lose values | timeline hook | **verify**: confirm `event_type` union unchanged; regen if it shifts |

Frontend (separate later work): VAT/binder/annual-report audit hooks + components switch to the generic `/audit/{type}/{id}`; regenerate `generated.ts` (known gotcha: `gen:types` overwrites in place — back up first).

## 15. Staged execution plan

**Phase 0 — Baseline & safety.** Run full pytest (`.venv` + `APP_ENV=test JWT_SECRET=x`), snapshot `openapi.json`. Acceptance: suite green; counts in this doc reconfirmed.

**Phase 1 — Schema/model upgrade + new migration.** Files: `audit_entity_audit_log.py`; **new** alembic migration (do not touch initial) that (a) alters `entity_audit_logs`: old/new/metadata_json → JSONB, add `actor_type`, `metadata_json`, make `performed_by` nullable, add 4 indexes, and (b) `DROP TABLE` `vat_audit_logs`, `annual_report_status_history`, `binder_lifecycle_logs`, `binder_intake_edit_logs`, `signature_audit_events`. Explicit `DROP TYPE` **only for Postgres enum types owned exclusively by these dropped tables** — do **not** drop enums shared with surviving tables (e.g. `AnnualReportStatus` is shared with `annual_reports` and must stay; the `from_status`/`to_status` columns reference it but don't own it). Phase 0 audits which enums each dropped table owns vs shares before the migration is written; `audit_constants.py` full ENTITY_*/ACTION_* + expanded `ALLOWED_READ_ENTITY_TYPES`. Tests: writer + endpoint. Acceptance: migration up/down clean on Postgres + SQLite; model + constants land.

**Phase 2 — Writer/repository upgrade.** `EntityAuditWriter` (+record_action/record_external_action, actor_type, metadata_json); `EntityAuditLogRepository` (+client-context queries). Update existing client/business/charge/annual-report writers to pass `metadata_json.client_record_id` + namespaced actions. Acceptance: existing writes carry metadata + actor_type; audit tests green.

**Phase 3 — Replace VAT audit.** Repoint 12 `append_audit` sites to EntityAuditWriter (`vat_work_item.*`/`vat_invoice.*`); delete model/repo/schema/route/seed. Rewrite VAT audit test → generic route. Acceptance: `/audit/vat_work_item/{id}` returns history; VAT suite green.

**Phase 4 — Replace AnnualReportStatusHistory.** 4 `append_status_audit_entry` sites → `annual_report.status_changed`. Delete model/repo/route/schema. Acceptance: status history via generic route + timeline.

**Phase 5 — Replace binder lifecycle + intake logs.** 9 `_append_log` → semantic `binder.*`; 2 `edit_intake` → `binder_intake.updated`. Delete both models/repos, `BinderAuditService`, `/binders/{id}/audit` + schema, binder seed lifecycle/intake writes. Update dashboard (drop `_serialize_binder`). Acceptance: dashboard + binder audit off EntityAuditLog.

**Phase 6 — Replace SignatureAuditEvent.** 10 `append_audit_event` → `signature_request.*` (signer events via `record_external_action`, forensic metadata). Delete model + mixin + schemas; update `TimelineRepository.list_signature_lifecycle_events`. **Verify no frontend signature-audit view depends on typed columns first.** Acceptance: signature timeline intact; forensic fields preserved.

**Phase 7 — Rebuild timeline & dashboard.** Apply the event-source registry (§4): re-point the 3 streams and finalize the dashboard EntityAuditLog-only path **first**; add timeline tests asserting each category resolves to exactly one source; **remove `_DEDUP_ACTIONS` only once those tests prove zero duplicates** across every category. Acceptance: registry covers all categories, no duplicate events, `_DEDUP_ACTIONS` gone, timeline/dashboard suites green.

**Phase 8 — Add missing audit writes by priority.** Critical (legal_entities, persons, advance_payment create/update, vat_invoices, documents, annual-report child lines, signature), then high/medium (authority_contacts, correspondence, notes, tasks, binders/handovers, invoices), then low/conditional (notifications, reminders, tax_calendar, deadline_rules). Add `actor_id` params where missing (§9). Acceptance: each tier writes EntityAuditLog with required metadata.

**Phase 9 — Seeds & tests.** Update 3 seed sites + orchestrator to emit EntityAuditLog (drop old-table builders). Full suite green.

**Phase 10 — OpenAPI/frontend regen.** Regenerate `openapi.json` + `generated.ts` (back up generated.ts first); update frontend audit hooks to generic endpoint; verify dashboard/timeline schemas (§14). Acceptance: contract sync clean; frontend types compile.

### Final acceptance (spec §13)
Only `EntityAuditLog` + `UserAuditLog` remain; old audit tables/models/repos/schemas/seed removed via new migration; JSONB for old/new/metadata; every spec-§8 business mutation + selected evidence events write EntityAuditLog; auth/security/admin-access only in UserAuditLog; timeline/dashboard work with one source per category and no `_DEDUP_ACTIONS`; no raw action strings in services; audit/vat/binders/annual_reports/signature/users/timeline/dashboard tests pass.
