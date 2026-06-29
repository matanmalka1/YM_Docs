# Audit Refactor — Phase 0 Report (Baseline & Exact Inventory)

Date: 2026-06-29
Scope: **inventory + baseline only.** Outputs: this report + `docs/audit-refactor-progress.md`.

## STATUS: BLOCKED — pending inventory corrections (revised after Phase 0 review)

Phase 0 review found the first draft incomplete/inaccurate. This revision corrects: write-site counts (now include direct `EntityAuditWriter.append()` + charge audit helper), baseline provenance (which checks actually ran in Phase 0 vs external), a disclosed **state mutation** from an authenticated health check, exact per-table cleanup DDL, persisted action names + raw-string inventory, an un-grouped registry/scope/visibility matrix, a full actor inventory, and complete legacy/repo/test reference lists. **Phase 1 must NOT start until these are accepted.**

## 0. Corrections applied in this revision

1. **Write-site counts corrected** — existing EntityAuditLog write sites are **30** (14 `record_*` + 13 direct `EntityAuditWriter.append()` in annual_reports + 3 charge via `record_charge_status_audit`), not 14. Business-target = **63**, total = **73** (was 47/57).
2. **Baseline provenance** — pytest (1517) and the frontend suite were **NOT run during Phase 0**; they are **external/user-confirmed**. Only vulture/pyright/ruff/audit-scripts were run in Phase 0.
3. **Health-check state mutation disclosed** — the authenticated health-check results (login, /me) were from an **external run, not Phase 0**; login **mutates `users.last_login_at` and writes a `UserAuditLog` login_success row** → not read-only. Phase-0/read-only health checks must be **unauthenticated** (/health, /info only).
4. **Cleanup DDL enumerated** per table; corrected the false claim that every legacy table has a timestamp index (**only `signature_audit_events` does**).
5. **Action matrix** now lists **persisted action names** (bare values today) + missing actions + a **raw action/entity string inventory**.
6. **Registry/scope/visibility** matrix un-grouped (one row per entity_type), with the clarification that `scope_to_active_clients_stmt` is a **deleted-client filter, not per-user authorization**, plus explicit deleted-history readability and per-role visibility.
7. **Actor inventory** covers every audited mutation + the `actor_display_name` source (currently absent everywhere).
8. **Full legacy/repo/test reference lists** added (§17).

## 1. Executive summary

- Baseline checks that **ran in Phase 0** are green (vulture, pyright, ruff check, ruff format --check, the 5 audit scripts). pytest + frontend are **external/user-confirmed**, not re-run here.
- **7 audit models** exist; target **2**. 5 legacy tables to merge & drop.
- Initial migration creates **42 tables** (confirmed).
- **Enum finding:** legacy audit tables own **no exclusive enum** (`action`/`event_type`/`actor_type`/`field_name` are `String`/`Text`); only `annualreportstatus` is involved and is **shared** with `annual_reports` → must stay. **Cleanup drops zero enum types.**
- **Serialization finding:** writers use `json.dumps` strings, one reader uses `json.loads` — must change with the JSONB switch.
- **Double-write finding:** annual_reports currently writes **both** EntityAuditLog (13 direct `append()`) **and** AnnualReportStatusHistory (3 `append_status_audit_entry`).
- **Status: BLOCKED** — see corrections above; recommendation deferred until accepted.

## 2. Baseline check results — provenance explicit

| Check | Ran in Phase 0? | Command | Result |
|---|---|---|---|
| Tests | **No (external)** | `pytest -q` | 1517 passed — **user-confirmed; not run in Phase 0** (background run cancelled per instruction) |
| Dead code | **Yes** | `vulture` | pass (rc=0) |
| Types | **Yes** | `pyright` | 0 errors / 0 warnings / 0 info |
| Format | **Yes** | `ruff format --check .` | pass (915 files) |
| Lint | **Yes** | `ruff check .` | pass |
| Migration chain | **Yes** | `check_migration_chain.py` | 0 findings (1 file, linear) |
| Role coverage | **Yes** | `check_role_coverage.py` | 0 findings (217 routes protected) |
| Pagination | **Yes** | `check_missing_pagination.py` | 0 findings (31 endpoints) |
| Unused routes | **Yes** | `check_unused_routes.py` | 4 findings (known external/manual; pre-existing) |
| Enum sync | **Yes** | `check_enum_sync.py` | 0 findings |
| Schema dump | **Yes** | `dump_schema.py` | 43 findings (informational; pre-existing) |
| Health (unauth) | **No (external)** | `/health`, `/info` | 200 — **user-confirmed; not run in Phase 0** |
| Health (auth) | **No (external) — STATE MUTATION** | login + `/me` | 200 — **NOT read-only**: login writes `last_login_at` + a UserAuditLog login_success row. Excluded from Phase-0 read-only scope; future read-only health checks must be unauthenticated. |
| OpenAPI sync | **No (external)** | `check_contract_sync.py` | in sync — user-confirmed |
| Frontend suite | **No (external)** | lint/typecheck/test/format/arch/arch:strict/knip | pass — **user-confirmed; not run in Phase 0** |

No check was modified to pass. `unused (4)` / `schema (43)` are pre-existing, unrelated, non-blocking.

## 3. Exact counts (corrected)

| Metric | Count | Note |
|---|---|---|
| Audit models | 7 | target 2 |
| Legacy tables to drop | 5 | §8 |
| Migration tables total | 42 | confirmed |
| Enum types to drop in cleanup | 0 | none exclusively owned (§7) |
| **EntityAuditLog write sites (existing)** | **30** | 14 `record_*` + 13 direct `append()` (annual_reports) + 3 charge via `record_charge_status_audit` |
| — `record_*` sites | 14 | clients/businesses/charges/annual_reports |
| — direct `EntityAuditWriter.append()` sites | 13 | annual_reports only (financial_line 6, vat_import 4, status 1, detail 1, annex 1) |
| — charge `record_charge_status_audit` sites | 3 | `charge_billing_service.py:107,135,169` → `charge_billing_audit.py:16` |
| Legacy-model write sites (to migrate in) | 33 | VAT 12 + signature 9 + binder lifecycle 7 + AR-status 3 + intake-edit 2 |
| **Business-target write sites total** | **63** | 30 + 33 |
| `UserAuditLog.log` write sites | 10 | auth/admin |
| **Total audit write sites** | **73** | 63 + 10 |
| Audit/history/timeline/dashboard/signature read routes | 9 | §5 |
| Seed builders writing legacy audit | 5 | §8 |
| Frontend audit consumer areas | 5 | §6 |
| Legacy-model reference files (non-test) | 22 | §17 |
| Legacy-repo usage files (non-test) | 13 | §17 |
| Test files referencing audit core | 29 | §17 |

## 4. Audit model inventory

| Model | File | Table | action field type | Target |
|---|---|---|---|---|
| EntityAuditLog | `app/audit/models/audit_entity_audit_log.py` | `entity_audit_logs` | `action: str` | KEEP + upgrade |
| UserAuditLog | `app/users/models/user_audit_log.py` | `user_audit_logs` | `action: AuditAction`, `status: AuditStatus` (enums) | KEEP (JSONB + snapshots) |
| VatAuditLog | `app/vat/models/vat_audit_log.py` | `vat_audit_logs` | `action: String` | DELETE → EntityAuditLog |
| AnnualReportStatusHistory | `app/annual_reports/models/annual_report_status_history.py` | `annual_report_status_history` | `from_status/to_status: Enum(annualreportstatus)` | DELETE → EntityAuditLog |
| BinderLifecycleLog | `app/binders/models/binder_lifecycle_log.py` | `binder_lifecycle_logs` | `field_name/old_value/new_value: String` | DELETE → EntityAuditLog |
| BinderIntakeEditLog | `app/binders/models/binder_intake_edit_log.py` | `binder_intake_edit_logs` | `field_name: String`, old/new `Text` | DELETE → EntityAuditLog (keep service) |
| SignatureAuditEvent | `app/signature_requests/models/signature_request.py` | `signature_audit_events` | `event_type/actor_type: String` | DELETE → EntityAuditLog |

## 5. Writer / reader / route inventory

**EntityAuditLog write sites (30):**
- `record_*` (14): clients (`client_create_service.py:216`, `client_update_service.py:70,105`, `client_lifecycle_service.py:32,55`), businesses (`business_service.py:101,177`, `business_lifecycle_service.py:24,37`), charges (`charge_billing_service.py:78,193`), annual_reports (`annual_report_create_service.py:150`, `annual_report_status_service.py:161`, `annual_report_service.py:70`).
- Direct `EntityAuditWriter(...).append()` (13, annual_reports): `annual_report_financial_line_service.py:94,131,153,197,234,256`; `annual_report_vat_import_service.py:173,191,227,324`; `annual_report_status_service.py:264`; `annual_report_detail_service.py:50`; `annual_report_annex_service.py:128`.
- Charge audit helper (3): `charge_billing_service.py:107,135,169` → `record_charge_status_audit` (`charge_billing_audit.py:16` `writer.append`).

**Legacy-model write sites (33):** VAT `append_audit` (12, §11-style, in vat services + `vat_work_item_metadata.py`), signature `append_audit_event` (9), binder `_append_log` (7), AR `append_status_audit_entry` (3), binder-intake edit (2). VAT routes the call through `vat_work_item_write_repository.append_audit:269` → `VatAuditLogRepository.append`.

**UserAuditLog `.log` (10):** `user_auth_service.py`, `user_management_service.py`, password-reset path.

**Read routes (9):**

| # | Route | File | Source today | After |
|---|---|---|---|---|
| 1 | `GET /audit/{entity_type}/{entity_id}` | `audit_routes.py:18` | EntityAuditLog | STAYS + expand via registry |
| 2 | `GET /users/audit-logs` | `user_routes_audit.py:24` | UserAuditLog | STAYS |
| 3 | `GET /vat/work-items/{id}/audit` | `vat_routes_queries.py:184` | VatAuditLog | DELETE → generic (Phase 3) |
| 4 | `GET /binders/{id}/audit` | `binder_routes_audit.py:26` | BinderLifecycleLog | DELETE → generic (Phase 5) |
| 5 | `GET /annual-reports/{id}/audit` | `annual_report_routes_status.py:129` | AnnualReportStatusHistory | DELETE → generic (Phase 4) |
| 6 | `GET /binders/{id}/intakes` | `binder_routes_audit.py:56` | BinderIntake | STAYS |
| 7 | `GET /clients/{id}/timeline` | `timeline_routes.py:19` | 9 categories | STAYS (re-pointed) |
| 8 | `GET /dashboard/overview` (`recent_activity`) | `dashboard_routes_overview.py` | EntityAuditLog + BinderLifecycleLog | CHANGES (Phase 5/7) |
| 9 | `GET /signature-requests/{id}` (`audit_trail`) | `signature_request_routes_advisor.py:101` | SignatureAuditEvent (embedded) | CHANGES — keep wrapper, re-source (Phase 6) |

**JSON serialization sites to change (Phase 1):** `audit_entity_audit_writer_service.py:143` (`json.dumps` in `_serialize_value`), `user_audit_log_repository.py:34` (`json.dumps`), `user_audit_log_service.py:83` (`json.loads`).

## 6. Frontend consumer inventory

| Backend change | Frontend consumers | Migrate in |
|---|---|---|
| Generic `/audit/{type}/{id}` gains fields | `src/features/audit/` (`api/audit.api.ts`, `api/contracts.ts`, `hooks/useEntityAuditTrail.ts`, `useEntityAuditTrailSection.ts`, `components/AuditTrailTable.tsx`, `EntityAuditTrailSection.tsx`); used by `BusinessDetailsCard.tsx`, `ChargeDetailDrawer.tsx` | Phase 2 (additive) + Phase 10 |
| `/vat/work-items/{id}/audit` deleted | `features/vatReports/` (`api/endpoints.ts`, `api/vatReports.api.ts`, `api/contracts.ts:VatAuditTrailResponse`, `hooks/useVatHistory.ts`, `components/detail/VatHistoryTab.tsx`) | **Phase 3** |
| `/annual-reports/{id}/audit` deleted | `features/annualReports/components/statusTransition/StatusAuditTimeline.tsx` | **Phase 4** |
| `/binders/{id}/audit` deleted | `features/binders/` (`components/drawer/BinderAuditSection.tsx`, `BinderDetailDrawer.tsx`, `api/endpoints.ts`, `api/queryKeys.ts`, `api/binders.api.ts`, `types.ts`) | **Phase 5** |
| `/signature-requests/{id}` `audit_trail` item shape | `features/signatureRequests/components/drawer/SignatureRequestAuditDrawer.tsx`, `api/contracts.ts`, `api/signatureRequests.api.ts` | **Phase 6** |
| Generated types | `src/types/generated.ts` — paths 3/4/5, schemas `EntityAuditTrailResponse`, `VatAuditTrailResponse`, embedded `audit_trail: SignatureAuditEventResponse[]` (~line 7135) | per-phase + final Phase 10 |

## 7. Enum ownership table

| Enum | Used by (table.column) | Shared with surviving table? | Safe to drop in cleanup? | Reason |
|---|---|---|---|---|
| `annualreportstatus` | `annual_report_status_history.from_status/to_status` **and** `annual_reports.status` | **YES** | **NO** | shared; dropping breaks `annual_reports` |
| `auditaction`, `auditstatus` | `user_audit_logs.action/status` | n/a (surviving) | NO | belongs to surviving UserAuditLog |
| (none) | `vat_audit_logs.action` = `String` | — | n/a | no enum |
| (none) | `binder_lifecycle_logs` / `binder_intake_edit_logs` = `String`/`Text` | — | n/a | no enum |
| (none) | `signature_audit_events.event_type/actor_type` = `String` | — | n/a | no enum |

**Cleanup migration drops zero enum types.**

## 8. Table inventory + exact cleanup DDL (per table)

42 tables (confirmed). 5 legacy audit tables to drop, with exact recreate-on-downgrade requirements (from `3e2669e69e32_initial.py`):

**`vat_audit_logs`** — cols: `id` PK int autoinc; `work_item_id` int NOT NULL; `performed_by` int NOT NULL; `action` String NOT NULL; `old_value` Text NULL; `new_value` Text NULL; `note` Text NULL; `invoice_id` int NULL; `performed_at` DateTime NOT NULL. FKs: `work_item_id→vat_work_items.id`, `performed_by→users.id`, `invoice_id→vat_invoices.id ON DELETE SET NULL`. PK: `id`. Indexes: `ix_vat_audit_logs_invoice_id(invoice_id)`, `ix_vat_audit_logs_work_item_id(work_item_id)`. **No timestamp index.**

**`annual_report_status_history`** — cols: `id` PK; `annual_report_id` int NOT NULL; `from_status` Enum(`annualreportstatus`) NULL; `to_status` Enum(`annualreportstatus`) NOT NULL; `changed_by` int NOT NULL; `note` Text NULL; `occurred_at` DateTime NOT NULL. FKs: `annual_report_id→annual_reports.id`, `changed_by→users.id`. PK: `id`. Indexes: `ix_annual_report_status_history_annual_report_id(annual_report_id)`. **No timestamp index.** Note: uses the **shared** `annualreportstatus` enum — do NOT drop it on cleanup.

**`binder_lifecycle_logs`** — cols: `id` PK; `binder_id` int NOT NULL; `field_name` String NOT NULL; `old_value` String NOT NULL; `new_value` String NOT NULL; `changed_by_user_id` int NOT NULL; `changed_at` DateTime NOT NULL; `notes` Text NULL. FKs: `binder_id→binders.id`, `changed_by_user_id→users.id`. PK: `id`. Indexes: `ix_binder_lifecycle_logs_binder_id(binder_id)`. **No timestamp index.**

**`binder_intake_edit_logs`** — cols: `id` PK; `intake_id` int NOT NULL; `field_name` String NOT NULL; `old_value` Text NULL; `new_value` Text NULL; `changed_by` int NOT NULL; `changed_at` DateTime NOT NULL. FKs: `intake_id→binder_intakes.id`, `changed_by→users.id`. PK: `id`. Indexes: `ix_binder_intake_edit_logs_intake_id(intake_id)`. **No timestamp index.**

**`signature_audit_events`** — cols: `id` PK; `signature_request_id` int NOT NULL; `event_type` String NOT NULL; `actor_type` String NOT NULL; `actor_id` int NULL; `actor_name` String NULL; `ip_address` String NULL; `user_agent` String NULL; `notes` Text NULL; `occurred_at` DateTime NOT NULL. FKs: `signature_request_id→signature_requests.id`. PK: `id`. Indexes: `idx_sig_audit_occurred(occurred_at)` **(timestamp index — only this table has one)**, `ix_signature_audit_events_signature_request_id(signature_request_id)`.

**Correction to first draft:** the claim "each table indexes its parent FK + the timestamp column" was wrong — only `signature_audit_events` has a timestamp index; the other four index only their parent FK.

## 9. Binding action matrix — persisted names (corrected)

Today persisted `action` values are **bare** (e.g. `"created"`, `"status_changed"`), namespaced only by `entity_type`. The plan target is **rich semantic** (`client.created`). The matrix below shows **current persisted constant/value → target action**. (Still requires sign-off; some current values are raw strings — see §9a.)

| Domain | entity_type | Current persisted action(s) (constant = value) | Target action | metadata_json | actor_type |
|---|---|---|---|---|---|
| clients | client | `ACTION_CREATED`="created", `ACTION_UPDATED`="updated", `ACTION_DELETED`="deleted", `ACTION_RESTORED`="restored", **`ACTION_ENTITY_TYPE_CHANGED`="entity_type_changed"** | client.created/updated/deleted/restored/entity_type_changed | client_record_id | user |
| businesses | business | created/updated/deleted/restored | business.* | client_record_id, business_id | user |
| charges | charge | **`ACTION_CREATED`="created"** (issue), **`ACTION_DELETED`="deleted"** (mark-paid path), `ACTION_ISSUED`="issued", `ACTION_PAID`="paid", `ACTION_CANCELED`="canceled" | charge.issued/paid/canceled (+ created/deleted reconciled) | client_record_id, business_id | user |
| annual_reports | annual_report | `ACTION_STATUS_CHANGED`, `ACTION_CREATED`, `ACTION_ANNUAL_REPORT_DETAIL_UPDATED`, **`ACTION_ANNUAL_REPORT_DEADLINE_UPDATED`**, `ACTION_METADATA_UPDATED` | annual_report.created/updated/status_changed/submitted/deleted | client_record_id, tax_year | user |
| AR child: income | annual_report | `ACTION_INCOME_ADDED/UPDATED/DELETED` | annual_report.income_line_added/updated/deleted | + section, line_id | user |
| AR child: expense | annual_report | `ACTION_EXPENSE_ADDED/UPDATED/DELETED` | annual_report.expense_line_added/updated/deleted | + section, line_id | user |
| AR child: annex | annual_report | **`ACTION_ANNEX_LINE_ADDED/UPDATED/DELETED`** | annual_report.annex_line_added/updated/deleted | + schedule_id, line_number | user |
| vat_work_items | vat_work_item | `ACTION_WORK_ITEM_CREATED_PENDING`, `ACTION_MATERIAL_RECEIVED`, `ACTION_STATUS_CHANGED`, `ACTION_OVERRIDE`="vat_override", `ACTION_FILED`, `ACTION_METADATA_UPDATED`, `ACTION_WORK_ITEM_DELETED` | vat_work_item.created/status_changed/filed/amount_overridden/deleted | client_record_id, period, tax_year | user |
| vat_invoices | vat_invoice | `ACTION_INVOICE_ADDED/UPDATED/DELETED` | vat_invoice.created/updated/amount_changed/deleted | client_record_id, vat_work_item_id, invoice_number | user |
| binders | binder | RAW field changes (`field_name="location_status"/"capacity_status"`) | binder.marked_full/reopened/marked_ready_for_handover/reverted_ready/handed_over | client_record_id, binder_id, field_name | user |
| binder_intakes | binder_intake | RAW field changes (`field_name=...`) | binder_intake.received/updated | client_record_id, binder_id, field_name | user |
| signature_requests | signature_request | RAW `event_type`="created/sent/viewed/signed/declined/canceled/expired/annual_report_signed" | signature_request.* | per §8a forensic | user/external_signer/system |
| legal_entities/persons/links/authority_contacts/notes/advance_payments/invoices/binder_handovers/tasks/correspondence/notifications/reminders/tax_calendar/deadline_rules | resp. | **NO audit today** (new) | new actions per §9 (plan) | client_record_id (where applicable) | user/system |

Missing-action items explicitly recovered (were absent from first draft): `client.entity_type_changed`, `charge.created`/`charge.deleted`, `annual_report.deadline_updated`, `annual_report.annex_line_added`.

### 9a. Raw action / entity string inventory

- **Action constants (audit_constants.py):** ENTITY_BUSINESS/CLIENT/CHARGE/ANNUAL_REPORT; ACTION_CREATED/UPDATED/DELETED/RESTORED/ENTITY_TYPE_CHANGED/ISSUED/PAID/CANCELED/STATUS_CHANGED/ANNUAL_REPORT_DETAIL_UPDATED/ANNUAL_REPORT_DEADLINE_UPDATED/ANNEX_LINE_ADDED/UPDATED/DELETED/INCOME_ADDED/UPDATED/DELETED/EXPENSE_ADDED/UPDATED/DELETED.
- **VAT action constants (vat_constants.py):** ACTION_WORK_ITEM_CREATED_PENDING/MATERIAL_RECEIVED/STATUS_CHANGED/INVOICE_ADDED/INVOICE_DELETED/INVOICE_UPDATED/OVERRIDE(="vat_override")/FILED/METADATA_UPDATED/WORK_ITEM_DELETED.
- **RAW strings (NOT constants):** signature `event_type` = `created/sent/viewed/signed/declined/canceled/expired/annual_report_signed`; signature `actor_type` = `advisor/signer/system` (`signature_request_*_service.py`); binder `field_name` = `location_status/capacity_status` (`binder_lifecycle_service.py`); binder-intake `field_name` (raw). These RAW strings must be promoted to constants during the refactor (spec §13: no raw action strings in services).
- **Inconsistency to resolve:** two `ACTION_STATUS_CHANGED` definitions exist (audit_constants + vat_constants) with the same value `"status_changed"`; namespacing to `vat_work_item.status_changed` vs `annual_report.status_changed` resolves it.

## 10. AuditEntityRegistry preparation matrix — un-grouped, per entity_type

**Authorization model clarification (important):** the system has two roles, `UserRole.ADVISOR` and `UserRole.SECRETARY`. **`scope_to_active_clients_stmt` is a soft-deleted-client *filter*, NOT per-user authorization** — it excludes deleted clients from active listings; it does not grant/deny by user. There is **no per-user ownership model**. So registry "scope" = (a) entity exists, (b) firm-level vs client-scoped, (c) deleted-history readability. Per-role visibility below encodes only what exists today (route auth + the deleted-client filter); it invents nothing.

**Deleted-history readability (explicit):** audit history must remain readable after the entity is soft- or hard-deleted (audit is the record of what happened). Therefore the registry's `resolve_scope` must **bypass the active-client filter for audit reads** and resolve scope with `include_deleted=True` (soft) or from `metadata_json.client_record_id` (hard). The deleted-client filter applies to *live listing*, not to *audit history*.

| entity_type | model / table | resolve_scope | allowed_roles | scope_policy | deleted_entity_policy | sensitive? | per-role forensic visibility |
|---|---|---|---|---|---|---|---|
| client | ClientRecord / client_records | self → {id} | ADVISOR, SECRETARY | client-scoped | soft: live+include_deleted; hard: audit metadata | no | n/a both roles |
| business | Business / businesses | legal_entity→client | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| legal_entity | LegalEntity / legal_entities | links→client(s) (may be multi) | ADVISOR, SECRETARY | client-scoped (union) | soft/hard | no | n/a |
| person | Person / persons | links→client(s) (multi) | ADVISOR, SECRETARY | client-scoped (union) | soft/hard | no | n/a |
| person_legal_entity_link | PersonLegalEntityLink | legal_entity→client | ADVISOR, SECRETARY | client-scoped | hard (link delete) via audit metadata | no | n/a |
| authority_contact | AuthorityContact | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| note | EntityNote | entity_type/id→client | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| advance_payment | AdvancePayment | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| charge | Charge | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| invoice | Invoice | charge→client | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| vat_work_item | VatWorkItem | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| vat_invoice | VatInvoice | work_item→client | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| annual_report | AnnualReport | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| binder | Binder | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| binder_intake | BinderIntake | binder→client | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| binder_handover | BinderHandover | client_record_id | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| document | PermanentDocument | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| signature_request | SignatureRequest | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard (hard via audit metadata) | **YES** | **OPEN — Phase 2 decision:** today both roles can open the signature drawer (no per-role gate exists). Default proposal: ip/user_agent/content_hash visible to both ADVISOR and SECRETARY (preserving current behavior), redaction applied only if a stricter rule is approved. Must be confirmed before Phase 6. |
| task | Task | client_record_id | ADVISOR, SECRETARY | client-scoped | soft/hard | no | n/a |
| notification | Notification | client_record_id | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| reminder | Reminder | source→client | ADVISOR, SECRETARY | client-scoped | hard via audit metadata | no | n/a |
| tax_calendar | TaxCalendarEntry | firm-level | ADVISOR, SECRETARY | firm-level (no client filter) | n/a | no | n/a |
| deadline_rule | DeadlineRule | firm-level | ADVISOR, SECRETARY | firm-level | n/a | no | n/a |

Scope cases covered: firm-level, one client, multi-client (legal_entity/person), soft-deleted, hard-deleted (audit-metadata). **Open item:** the only unresolved visibility decision is the signature_request sensitive forensic fields per role (above) — flagged for Phase 2 sign-off.

## 11. Signature audit replacement inventory

- **Writers (9):** `signature_request_creation_service.py:114,122`; `signature_request_signer_service.py:32,53,88,126`; `signature_request_admin_service.py:45,68`; `signature_request_service.py:79`. Event types (raw): created/sent/viewed/signed/declined/canceled/expired/annual_report_signed; actor_type advisor/signer/system.
- **Readers:** `TimelineRepository.list_signature_lifecycle_events:43-58` (joins SignatureRequest×SignatureAuditEvent, filters `_SIGNATURE_LIFECYCLE_TYPES`) → `timeline_client_aggregator.py:32` → builder `timeline_client_builders.py:87`; **embedded** `audit_trail` via `signature_request_service.get_audit_trail` → `repo.list_audit_events` → `build_with_audit` (`signature_request_response_builder.py:41-45`) on `GET /signature-requests/{id}`.
- **Frontend:** `SignatureRequestAuditDrawer.tsx` ← `audit_trail: SignatureAuditEventResponse[]` (`generated.ts:~7135`).
- **Replacement (Phase 6):** keep `SignatureRequestWithAuditResponse`; new embedded item `{action, actor_type, actor_display_name, performed_at, note, +forensic from metadata_json}`. Items fetched via `AuditTrailService`, passed into `build_with_audit` (already accepts items — no repo dependency).
- **Forensic mapping (§8a):** event_type→`signature_request.<event>`; actor advisor→user / signer→external_signer / system→system; `actor_name→actor_display_name`; `ip_address/user_agent/content_hash/signed_document_key→metadata_json` per action.
- **Seed:** `signature_requests.py:229`.

## 12. Timeline / dashboard inventory

**Timeline sources:** client created (live), business changed (EntityAuditLog), charge created/issued/paid+invoice (live), annual_report status (AnnualReportStatusHistory via `timeline_repository:63`), binder received/handed_over (live), binder lifecycle (BinderLifecycleLog via `timeline_service`), signature lifecycle (SignatureAuditEvent via `timeline_repository:43`), document uploaded (live), notifications (live). Dedup: `_DEDUP_ACTIONS` in `timeline_audit_aggregator.py`.

**Moving to EntityAuditLog:** annual_report status (P4), binder lifecycle (P5), signature (P6). VatAuditLog + BinderIntakeEditLog are not read by timeline.

**Dashboard (`dashboard_recent_activity_service.py`):** EntityAuditLog `list_recent` + BinderLifecycleLog `list_recent` (line 95); `_serialize_binder` (166), `_binder_label` (179), dual-timestamp `_timestamp` (212). After P5: EntityAuditLog-only.

**Duplicate risk / registry gaps:** `binder.handed_over` exists as both a live builder and a future audit action → single source required; `_DEDUP_ACTIONS` removed only after single-source tests prove zero duplicates (P7).

## 13. Actor availability inventory (every audited mutation)

`actor_display_name` source: **no service currently passes a display name.** Two sourcing options (decide Phase 2): (A) thread `actor_display_name=current_user.full_name` from the route alongside `actor_id`; (B) writer looks up `users.full_name` by `performed_by` at write time (single source; null for system/external_signer where the writer uses a system label / signer name). Recommendation: **B for user actors** (one place), explicit label for system/external_signer.

| Domain | Function(s) | Has actor id? | Param | actor_type | Fix |
|---|---|---|---|---|---|
| clients | create/update/delete/restore | yes | `created_by`/`actor_id` | user | reuse |
| businesses | create/update/delete/restore | yes | `created_by`/`actor_id` | user | reuse |
| legal_entities | (mutated via client/business create/update) | **partial** | none dedicated | user | **thread actor from owning route** |
| persons | (mutated via client/business) | **partial** | none dedicated | user | **thread actor** |
| person_legal_entity_links | link create/delete (via client/business) | **partial** | none dedicated | user | **thread actor** |
| authority_contacts | `add_contact:19` / `update_contact:45` | **NO** | — | user | **add `actor_id`** |
| authority_contacts | `delete_contact:80` | yes | `actor_id` | user | reuse |
| entity_notes | add/update/delete | yes | `created_by`/`actor_id` | user | reuse |
| advance_payments | `create_payment_for_client:117` / `update_payment_for_client:193` | **NO** | — | user | **add `actor_id`** |
| advance_payments | `delete_payment_for_client:238` | yes | `actor_id` | user | reuse |
| charges | issue/pay/cancel | yes | `created_by`/`issued_by`/`paid_by`/`canceled_by` | user | reuse |
| invoices | create | yes (model fields) | — | user | wire to writer |
| vat_work_items | create/status/filed/override/delete | yes | `created_by`/`performed_by` | user | reuse |
| vat_invoices | add/update/delete | yes | `created_by` | user | reuse |
| annual_reports | create/status/detail/deadline | yes | `created_by`/`changed_by`/`actor_id` | user | reuse |
| AR child rows | income/expense/annex/schedule | yes | parent actor | user | reuse |
| binders | lifecycle transitions | yes | `changed_by_user_id`/`actor_id` | user | reuse |
| binder_intakes | `receive` / edit | yes | `received_by`/`changed_by` | user | reuse |
| binder_handovers | create | yes | `created_by` | user | reuse |
| permanent_documents | `upload_document:95` | yes | `uploaded_by` | user | wire to writer |
| permanent_documents | approve/reject/`replace_document:309` | yes | actor fields | user | wire to writer |
| permanent_documents | `delete_document:302` | **NO** | — | user | **add `actor_id`** |
| signature_requests | advisor actions (create/sent/cancel) | yes | current_user | user | reuse |
| signature_requests | signer actions (viewed/signed/declined) | n/a (external) | — | external_signer | performed_by=None, name→actor_display_name |
| signature_requests | system (expired/annual_report_signed) | n/a | — | system | performed_by=None, system label |
| tasks | create/assign/complete/cancel/delete | yes | `created_by_user_id`/completed/canceled fields | user | reuse |
| correspondence | create/update/delete | yes | `created_by` | user | reuse |
| notifications | send | yes/system | `triggered_by` | user/system | reuse; system→None |
| reminders | created/fired/failed | system (fired) / user (manual) | `created_by_user_id` | system/user | system label for fired |
| tax_calendar | generate/manual edit | system/user | — | system/user | system label for generation |
| deadline_rules | edit (if UI) | user | — | user | route actor |

**Missing-actor fixes (5 surfaces, confirmed):** advance_payment create + update, authority_contact add + update, permanent_document delete. **Plus actor-threading needed** for legal_entity / person / person_legal_entity_link mutations (no dedicated actor param; flow through client/business routes).

## 14. Blockers

This report itself was **blocked** on the inaccuracies now corrected (§0). With those corrections, the remaining **open items requiring decision before the relevant phase** are:
1. **Signature sensitive forensic visibility per role** (§10) — Phase 2/6.
2. **`actor_display_name` sourcing** (A vs B) (§13) — Phase 2.
3. **`charge.created`/`charge.deleted` vs `record_charge_status_audit`** action reconciliation (§9) — Phase 2/3.
4. **AR double-write** (EntityAuditLog + AnnualReportStatusHistory) collapse without losing data semantics — Phase 4.
No baseline failure exists.

## 15. Risks (carried into implementation)

Signature forensic/legal parity (embedded drawer + timeline); timeline duplicate events; JSON serializer/reader switch + portable `JSON().with_variant(JSONB)`; Phase-1 NOT NULL sequencing (1a→1b); fail-safe downgrade; shared `annualreportstatus` enum; layering (service owns scope/authz/redaction); frontend per-phase migration; raw action strings → constants; AR double-write collapse.

## 16. Recommendation

**Deferred — Phase 0 is BLOCKED** pending acceptance of the corrections in §0 and decisions on the four open items in §14. Phase 1 must NOT start until this revised inventory is accepted. Once accepted with the counts in §3 (30/63/73), the enum result (zero drops), the per-table cleanup DDL (§8), and the actor/registry matrices, Phase 1 (schema add/alter 1a→1b + serializer/reader + writer actor support, no legacy drops) is technically ready.

## 17. Full reference lists (requested by Phase 0)

**Legacy-model reference files (non-test, 22):** annual_reports/models/__init__.py, annual_reports/models/annual_report_status_history.py, annual_reports/repositories/annual_report_status_audit_repository.py, binders/models/binder_intake_edit_log.py, binders/models/binder_lifecycle_log.py, binders/repositories/binder_intake_edit_log_repository.py, binders/repositories/binder_lifecycle_log_repository.py, binders/services/binder_audit_service.py, dashboard/services/dashboard_recent_activity_service.py, seed/builders/demo/binders.py, seed/builders/demo/reports.py, seed/builders/demo/signature_requests.py, seed/builders/demo/vat.py, signature_requests/models/signature_request.py, signature_requests/repositories/signature_request_audit.py, signature_requests/repositories/signature_request_repository.py, timeline/repositories/timeline_repository.py, vat/api/vat_routes_queries.py, vat/models/vat_audit_log.py, vat/repositories/vat_audit_log_repository.py, vat/repositories/vat_work_item_write_repository.py, vat/schemas/vat_audit.py.

**Legacy-repo usage files (non-test, 13):** annual_reports/repositories/annual_report_repository.py, annual_reports/repositories/annual_report_status_audit_repository.py, binders/repositories/binder_intake_edit_log_repository.py, binders/repositories/binder_lifecycle_log_repository.py, binders/services/binder_audit_service.py, binders/services/binder_intake_edit_service.py, binders/services/binder_lifecycle_service.py, dashboard/services/dashboard_recent_activity_service.py, signature_requests/repositories/signature_request_audit.py, signature_requests/repositories/signature_request_repository.py, timeline/services/timeline_service.py, vat/repositories/vat_audit_log_repository.py, vat/repositories/vat_work_item_write_repository.py.

**Seed builders writing legacy audit (5):** seed/orchestrator.py:214 (`create_vat_audit_logs`), seed/builders/demo/vat.py:465,483, seed/builders/demo/binders.py:173 (BinderLifecycleLog), :341,355 (BinderIntakeEditLog), seed/builders/demo/signature_requests.py:229 (SignatureAuditEvent), seed/builders/demo/reports.py:625 (AnnualReportStatusHistory).

**Test files referencing audit core (29):** tests/annual_reports/api/test_annual_report_audit.py, tests/annual_reports/api/test_annual_report_create_read_additional.py, tests/annual_reports/service/test_annual_report_generic_audit.py, tests/annual_reports/service/test_annual_report_query_service.py, tests/annual_reports/service/test_financial_audit_snapshots.py, tests/annual_reports/service/test_vat_import_service.py, tests/audit/test_audit_endpoint.py, tests/audit/test_audit_endpoint_filters.py, tests/audit/test_entity_audit_writer.py, tests/auth/api/test_logout_invalidation.py, tests/binders/api/test_binder_audit.py, tests/binders/api/test_binders.py, tests/binders/service/test_binder_intake_edit_service.py, tests/binders/service/test_binder_intake_vat_auto_advance.py, tests/binders/service/test_binder_lifecycle_service.py, tests/businesses/service/test_business_service_additional.py, tests/charges/service/test_billing_audit.py, tests/clients/service/test_entity_type_change_guard.py, tests/core/test_error_doc_coverage_matrix.py, tests/core/test_openapi_audit_paths.py, tests/regression/test_readonly_endpoints_no_side_effects.py, tests/signature_requests/api/test_signature_request_updated_at.py, tests/signature_requests/api/test_signature_requests.py, tests/signature_requests/api/test_signature_requests_cancel_and_client_list.py, tests/signature_requests/repository/test_signature_request_repository.py, tests/signature_requests/service/test_signature_requests.py, tests/timeline/service/test_timeline_operational_policy.py, tests/timeline/service/test_timeline_signature_lifecycle.py, tests/users/api/test_user_audit_logs.py, tests/users/services/test_audit_log_service.py, tests/users/services/test_auth_service_additional.py, tests/users/services/test_user_management_service_list_reset.py, tests/vat/api/test_vat_reports_audit.py, tests/vat/api/test_vat_reports_filing.py, tests/vat/api/test_vat_work_item_mutations.py, tests/vat/repository/test_vat_work_item_repository.py, tests/vat/service/test_vat_report_service_queries.py, tests/vat/service/test_vat_work_item_metadata.py.
