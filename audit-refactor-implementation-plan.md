# Audit Refactor — Implementation Plan

Status: **planning only**. This document changes no production code, migration, model, schema, seed, generated file, or frontend. DB is dev/seed-only → no data backfill, no dual-write, no backward compatibility, no legacy aliases.

Target end state: exactly **two** audit models.

- **EntityAuditLog** — every business mutation **plus selected evidence events** (state-machine / legal-evidence transitions such as `signature_request.viewed/signed/declined`, `annual_report.submitted`, `charge.paid`, `binder.handed_over`).
- **UserAuditLog** — auth/security/admin-access only (login, logout, password reset, user create/activate/deactivate/role-change).

`VatAuditLog`, `AnnualReportStatusHistory`, `BinderLifecycleLog`, `BinderIntakeEditLog`, `SignatureAuditEvent` are deleted and merged into `EntityAuditLog`.

> **Counts in this document are provisional.** Where a number appears it is the best current grep/scan reading, not a guarantee. **Phase 0 regenerates the exact inventory** (writers, readers, routes, tests, tables, enums) before any code is written. Do not treat counts as authoritative until Phase 0 confirms them.

### Locked decisions
- **Read routes — collapse to generic**, gated by a real authorization layer (§3a). Per-domain audit read routes are deleted; `GET /audit/{entity_type}/{entity_id}` becomes the single entity-audit read route — but only after the `AuditEntityRegistry` resolver/authorization plan (§3a) is in place. The **signature detail drawer is a special case** (§4a).
- **Action granularity — rich semantic.** Specific verbs + `domain.created/updated/deleted` for plain edits. Annual-report child operations keep rich semantic actions (§7a). Field diffs always in `old_value`/`new_value`.
- **Migration — staged, never a rewrite of the initial migration** (§1a). Phase 1 only adds/alters the two surviving tables; legacy tables are dropped only after their writers/readers/routes/tests are gone (one cleanup migration after Phases 3–7, or per-phase).

---

## Target models

**EntityAuditLog** (`app/audit/models/audit_entity_audit_log.py`, table `entity_audit_logs`):
```
id, entity_type:str, entity_id:int,
action:str,                       # rich semantic "domain.action"
performed_by:int|None (FK users.id, nullable),
actor_type:str (NOT NULL),        # user | system | external_signer
actor_display_name:str|None,      # immutable actor snapshot (§5)
old_value:JSONB|None, new_value:JSONB|None, metadata_json:JSONB|None,
note:str|None, performed_at:datetime
```
Delta vs today: `old_value`/`new_value` Text→JSON; **add** `actor_type` (NOT NULL), `actor_display_name` (nullable), `metadata_json` (JSON); `performed_by` → Integer FK `users.id`, **nullable** (was non-null). Indexes: `(entity_type, entity_id, performed_at)`, `(action, performed_at)`, `(performed_by, performed_at)`, `(performed_at)`, plus the client-context expression index (§8b).

**Model column type (portability).** Model-level JSON columns use the portable form `JSON().with_variant(JSONB, "postgresql")` (or the repo's equivalent convention) — **not** a bare `JSONB` column type, which would break SQLite `create_all` in dev/tests. PostgreSQL gets JSONB at runtime; SQLite gets JSON/TEXT. Alembic migrations stay PostgreSQL-specific with explicit `USING` casts (§1b).

**UserAuditLog** (`app/users/models/user_audit_log.py`, table `user_audit_logs`): keep. Changes: `metadata_json` Text→JSONB; **add `actor_display_name` + `target_display_name`** nullable snapshots (§5). Keep `action`/`status` as the existing closed enums (`AuditAction`/`AuditStatus`).

**Decision — user activate/deactivate stays UserAuditLog-only.** No dual-write to `EntityAuditLog`; **no `ENTITY_USER` constant** (§6). Auditing user-entity changes in `EntityAuditLog` is out of scope.

---

## 1. Audit models inventory (provisional — Phase 0 confirms)

7 models found (target 2).

| Model | File | Table | Writer sites (provisional) | Readers | API | Target |
|---|---|---|---|---|---|---|
| **EntityAuditLog** | `app/audit/models/audit_entity_audit_log.py` | `entity_audit_logs` | `EntityAuditWriter.record_*` (14) | `AuditTrailService`, timeline, dashboard | `GET /audit/{type}/{id}` | **KEEP + upgrade** |
| **UserAuditLog** | `app/users/models/user_audit_log.py` | `user_audit_logs` | `AuditLogService.log` (10) | `AuditLogService.list_logs` | `GET /users/audit-logs` | **KEEP** (JSONB + snapshots) |
| **VatAuditLog** | `app/vat/models/vat_audit_log.py` | `vat_audit_logs` | `work_item_repo.append_audit` (12) | `VatReportService.get_audit_trail_enriched` | `GET /vat/work-items/{id}/audit` | **DELETE → EntityAuditLog** |
| **AnnualReportStatusHistory** | `app/annual_reports/models/annual_report_status_history.py` | `annual_report_status_history` | `append_status_audit_entry` (**3**) | `AnnualReportService.get_report_audit`, timeline | `GET /annual-reports/{id}/audit` | **DELETE → EntityAuditLog** |
| **BinderLifecycleLog** | `app/binders/models/binder_lifecycle_log.py` | `binder_lifecycle_logs` | `BinderLifecycleService._append_log` (**7**) | `BinderAuditService`, timeline, dashboard | `GET /binders/{id}/audit` | **DELETE → EntityAuditLog** |
| **BinderIntakeEditLog** | `app/binders/models/binder_intake_edit_log.py` | `binder_intake_edit_logs` | `BinderIntakeEditService` (2) | none (no read route) | written via `PATCH /binders/{id}/intakes/{id}` | **DELETE table/repo → EntityAuditLog; KEEP `BinderIntakeEditService`** (§10b) |
| **SignatureAuditEvent** | `app/signature_requests/models/signature_request.py` | `signature_audit_events` | `append_audit_event` (**9**) | `TimelineRepository.list_signature_lifecycle_events` **+ advisor detail `audit_trail`** | embedded in `GET /signature-requests/{id}` (§4a) | **DELETE → EntityAuditLog** |

Deleted repos/services/schemas: `VatAuditLogRepository` + `vat/schemas/vat_audit.py`; `AnnualReportStatusAuditRepository` + `AnnualReportAuditEntry`/`AnnualReportAuditListResponse`; `BinderLifecycleLogRepository` + `BinderAuditService` + `BinderAuditEntry`/`BinderAuditResponse`; `BinderIntakeEditLogRepository` (but **keep `BinderIntakeEditService`**); `SignatureRequestAuditMixin` + `SignatureAuditEventResponse` (the per-item shape). **`SignatureRequestWithAuditResponse` is KEPT** and its `audit_trail` re-sourced from EntityAuditLog with a new embedded item schema (§4a).

### 1a. Migration ordering (correction #1)

- **The initial migration `alembic/versions/3e2669e69e32_initial.py` is never rewritten.**
- **Phase 1 migration** adds/alters **only** the two surviving tables (`entity_audit_logs`, `user_audit_logs`). It does **not** drop any legacy audit table.
- **Legacy tables are dropped only after their final writer/reader/route/test is replaced.** Either drop each table at the end of its own replacement phase (3–7) or — preferred — a **single cleanup migration after Phases 3–7** drops all five at once.
- **Migration round-trip target is PostgreSQL.** SQLite may be used for dev convenience, but **do not claim JSONB round-trip on SQLite** — SQLite stores JSON as TEXT. PostgreSQL is the authority for up+down round-trip in CI (Phase 0 + Phase 10).

### 1b. Migration details (correction #2)

Phase 1 is **safe-sequenced** so existing audit writes never break (correction #1). It pairs the schema change with the serializer/reader change (correction #2) and splits NOT-NULL enforcement into a final substep after writers are updated. Implement as **migration 1a + 1b** (two Alembic revisions in the same phase):

**Step A — schema + writer/reader (additive, no NOT NULL yet) [migration 1a]:**
- `entity_audit_logs.old_value`, `.new_value`: Text → JSONB via explicit PostgreSQL `USING`:
  `op.alter_column("entity_audit_logs", "old_value", type_=postgresql.JSONB(), postgresql_using="old_value::jsonb")` (same for `new_value`). Existing seed values are JSON strings; the cast succeeds on PostgreSQL.
- Add `metadata_json` JSONB nullable; add `actor_display_name` String nullable.
- Add `actor_type` with a **temporary `server_default="user"`** and **nullable** for now (existing rows + any in-flight writes stay valid).
- `performed_by`: make nullable.
- `user_audit_logs.metadata_json`: Text → JSONB (`USING metadata_json::jsonb`); add `actor_display_name`, `target_display_name` nullable.
- Add indexes from Target models + the §8b expression index.
- **Serializer/reader changes in the same step (correction #2):**
  - `EntityAuditWriter._serialize_value` stops calling `json.dumps`; it returns the **normalized dict/list/JSON value directly** (JSONB column stores the object). Keep `_normalize_value` (Decimal/date/enum → JSON-safe) but drop the surrounding `json.dumps`.
  - `UserAuditLogRepository.create` writes `metadata` as a **dict directly** (drop `json.dumps`).
  - Readers stop `json.loads` — but the two response schemas are **different shapes** (do not conflate):
    - **EntityAuditLog** — `EntityAuditLogResponse` (via `AuditTrailService`) types `old_value`/`new_value`/`metadata_json` as **JSON objects** (`dict | list | None`), not strings; plus the new `actor_type` + `actor_display_name` fields.
    - **UserAuditLog** — `UserAuditLogResponse` has **no** `old_value`/`new_value` and **no** `actor_type`; it exposes `metadata` as a **JSON object** (was a `json.loads`-ed string in `AuditLogService._to_dict`), plus the new `actor_display_name` + `target_display_name` snapshots.
  - Existing writers updated **in this step**:
    - EntityAuditLog writers (client/business/charge/annual-report — the 30 sites) pass `actor_type` + `actor_display_name`.
    - UserAuditLog writers (`user_auth_service.py`, `user_management_service.py`, password-reset) pass `actor_display_name` (+ `target_display_name` where a target user exists) — **not `actor_type`** (UserAuditLog has no such column). These snapshots must be captured so renames don't rewrite auth/admin history.
  - Add **JSON-object round-trip tests** (write dict → read dict; EntityAuditLog old/new/metadata; UserAuditLog metadata; response schemas show objects not strings).

**Step B — enforce NOT NULL (after writers populate it) [migration 1b]:**
- Once all writers pass `actor_type`, `op.alter_column("entity_audit_logs", "actor_type", nullable=False)` and **drop the temporary `server_default`** (`server_default=None`). New rows must pass `actor_type` explicitly (enforced by the §5a validation matrix in the writer).

**Phase-1 downgrade — clean, fail-safe (correction #6, Option A):**
- JSONB → Text reverse conversion with explicit cast on both tables: `op.alter_column(..., type_=sa.Text(), postgresql_using="old_value::text")` (and `new_value`, `metadata_json`); drop the added columns/indexes; restore `actor_type` default if reversing 1b.
- **`performed_by` restore is fail-safe, not silently asymmetric:** before re-imposing `NOT NULL`, the downgrade **asserts no rows have `performed_by IS NULL`**. If null-actor rows exist (system/external_signer), the **downgrade fails with a clear error message** rather than silently leaving the schema changed or destroying rows. This is a clean, deterministic round-trip: it either fully reverses or refuses with a explicit message. (Reversing 1b before 1a also restores `actor_type` nullable + default in the correct order.)

**Cleanup migration (after Phases 3–7):**
- `DROP TABLE` the five legacy audit tables.
- **Downgrade must fully recreate** each dropped table with its original columns, indexes, FKs, and constraints (copy definitions from `3e2669e69e32_initial.py`). A non-reversible downgrade is not acceptable.
- `DROP TYPE` **only for Postgres enum types owned exclusively by the dropped tables.** Do **not** drop shared enums — e.g. `AnnualReportStatus` is shared with the surviving `annual_reports` table (`from_status`/`to_status` reference it but don't own it) and must stay. Phase 0 enumerates, per dropped table, which enums it owns vs shares.

## 2. Call-site map (provisional)

Write call-sites (grep, excluding tests) — **to be reconfirmed in Phase 0**:

| Source | Sites |
|---|---|
| `EntityAuditWriter.record_*` (EntityAuditLog) | 14 |
| `work_item_repo.append_audit` → VatAuditLog | 12 |
| `append_audit_event` → SignatureAuditEvent | **9** |
| `BinderLifecycleService._append_log` → BinderLifecycleLog | **7** |
| `append_status_audit_entry` → AnnualReportStatusHistory | **3** |
| `BinderIntakeEditService` → BinderIntakeEditLog | 2 |
| `AuditLogService.log` → UserAuditLog (auth/admin) | 10 |

Read/aggregation paths: `AuditTrailService`, `VatReportService.get_audit_trail_enriched`, `BinderAuditService`, `AnnualReportService.get_report_audit`, `AuditLogService.list_logs`, timeline aggregator + `TimelineRepository` (status/lifecycle/signature), dashboard `RecentActivityService.build`, **signature advisor-detail `audit_trail`** (§4a). API: §3. Seed: `app/seed/builders/demo/vat.py`, `app/seed/builders/demo/binders.py`, `app/seed/orchestrator.py`. Tests: §13.

## 3. Audit/history/timeline routes

| # | Route | File | Source today | After |
|---|---|---|---|---|
| 1 | `GET /audit/{entity_type}/{entity_id}` | `app/audit/api/audit_routes.py:18` | EntityAuditLog | **STAYS + EXPANDS** behind `AuditEntityRegistry` authorization (§3a) |
| 2 | `GET /users/audit-logs` | `app/users/api/user_routes_audit.py:24` | UserAuditLog | **STAYS** |
| 3 | `GET /vat/work-items/{id}/audit` | `app/vat/api/vat_routes_queries.py:184` | VatAuditLog | **DELETE** → `/audit/vat_work_item/{id}` |
| 4 | `GET /binders/{id}/audit` | `app/binders/api/binder_routes_audit.py:26` | BinderLifecycleLog | **DELETE** → `/audit/binder/{id}` |
| 5 | `GET /annual-reports/{id}/audit` | `app/annual_reports/api/annual_report_routes_status.py:129` | AnnualReportStatusHistory | **DELETE** → `/audit/annual_report/{id}` |
| 6 | `GET /binders/{id}/intakes` | `app/binders/api/binder_routes_audit.py:56` | BinderIntake | **STAYS** |
| 7 | `GET /clients/{id}/timeline` | `app/timeline/api/timeline_routes.py:19` | 9 categories | **STAYS** (sources re-pointed, §4) |
| 8 | `GET /dashboard/overview` (`recent_activity`) | `app/dashboard/api/dashboard_routes_overview.py` | EntityAuditLog + BinderLifecycleLog | **CHANGES** — EntityAuditLog only (§5) |
| 9 | `GET /signature-requests/{id}` (`audit_trail`) | `app/signature_requests/api/signature_request_routes_advisor.py:101` | SignatureAuditEvent (embedded) | **CHANGES** — re-source `audit_trail` from EntityAuditLog (§4a) |

`PATCH /binders/{id}/intakes/{id}` continues to write via `BinderIntakeEditService`, now into EntityAuditLog (§10b).

### 3a. Generic audit route — AuditEntityRegistry, scope & authorization

Expanding `ALLOWED_READ_ENTITY_TYPES` is **not sufficient**. The generic route must **authorize before returning any audit rows**, using the **existing authorization model only** — no new permission model is introduced by this refactor.

**Authorization model (existing, do not invent):** the system has exactly two roles, `UserRole.ADVISOR` and `UserRole.SECRETARY` (`app/users/models/user.py`), and both may read the generic audit route. `scope_to_active_clients_stmt` is a live-list deleted-client filter, **not authorization**; audit-history reads bypass it so deleted history remains readable. There is no disallowed authenticated role and no per-user ownership model. Any new ownership/permission model (per-accountant ownership, record-level grants, etc.) is out of scope.

Introduce an `AuditEntityRegistry` (in `app/audit/`) keyed by `entity_type`, each entry a descriptor:

| Field | Meaning |
|---|---|
| `model` / `table` | the audited entity's SQLAlchemy model + table |
| repository-backed live resolver | existence/scope lookup against the live table with `include_deleted=True`; registry/service code does not execute SQL |
| `resolve_scope(entity_id, audit_rows) -> AuditScope` | service-orchestrated scope result from repository output when live, else immutable audit metadata (see below) |
| `sensitive` flag + `access_rule` | marks entity types whose audit carries restricted forensic/PII (e.g. `signature_request`); §16 governs read-time redaction + who may read, within the existing role model |

**`resolve_scope()` replaces a single `client_record_id` resolver.** It returns an `AuditScope`:
```
AuditScope:
    client_ids: set[int]          # owning client(s); empty if firm-level
    firm_level: bool              # True for firm-wide entities (tax_calendar, deadline_rule)
    entity_deleted: bool          # True if the live entity is soft- or hard-deleted
    resolved_from: "live_table" | "audit_metadata"
```
It must support: **zero client context / firm-level** (`firm_level=True`, empty `client_ids`); **one client**; **multiple clients** (`client_ids` with >1); **soft-deleted entities** (resolve from live row with `include_deleted=True`, `entity_deleted=True`); and **hard-deleted entities** (live row gone → resolve `client_ids` from immutable `metadata_json.client_record_id` on the audit rows, `resolved_from="audit_metadata"`, `entity_deleted=True`). **Hard-deleted audit history must remain readable** — it must not become inaccessible just because the live row was removed.

**Layering (correction #5).** Repositories do **DB access only**. **`AuditTrailService` owns** registry lookup, `resolve_scope`, authorization, redaction, and audit→response mapping. **Routes call `AuditTrailService`** and never touch the registry, repository, or scope logic directly. Other domains that need audit items (e.g. the signature response builder, §4a) also go **through `AuditTrailService`**, not through `EntityAuditLogRepository` directly.

**Service flow (`AuditTrailService.get_entity_audit_trail`):** validate `entity_type ∈ registry` → fetch audit rows → ask the repository-backed resolver for the live entity with `include_deleted=True` → if absent, resolve scope from immutable audit metadata → return 404 only when neither a live entity nor usable historical metadata exists → authorize the current role (both ADVISOR and SECRETARY are allowed) → apply the sensitive-data hook → map to response. `entity_deleted` appears once on the `EntityAuditTrailResponse` envelope, not on every item. The route injects `CurrentUser`, passes it to the service, and handles transport mapping only. No entity type is added without model/repository resolver/scope/sensitive definitions. Scope resolution is not per-user authorization, and audit history does not apply the active-client filter.

## 4. Timeline — one source of truth per category (correction #5)

`app/timeline/services/timeline_service.py` aggregates the categories in the registry below and is a **product timeline**, not a raw audit viewer. Files: `timeline_service.py`, `timeline_audit_aggregator.py` (`_DEDUP_ACTIONS`), `timeline_client_builders.py` (`entity_audit_changed_event`), `timeline/repositories/timeline_repository.py`, `timeline_labels.py`. Client anchor = `client_record_id` input.

Today a logical event can arrive from **both** a live builder and an EntityAuditLog row, patched by `_DEDUP_ACTIONS`. Replace that with an explicit **event-source registry** — one source per category:

| Category | Single source |
|---|---|
| client created | live builder (`client_created_event`) |
| business changed | EntityAuditLog `business.*` |
| charge created/issued/paid + invoice attached | live builders (`charge_*`, `invoice_attached`) |
| annual_report status changed / submitted | EntityAuditLog `annual_report.status_changed`/`submitted` |
| binder received / handed_over | live builders |
| binder lifecycle (marked_full/reopened/ready/reverted) | EntityAuditLog `binder.*` (excludes received/handed_over) |
| signature lifecycle (sent/viewed/signed/declined/canceled/expired) | EntityAuditLog `signature_request.*` |
| document uploaded | live `PermanentDocument` |
| notifications sent/failed | live `Notification` |

**Recommendation:** keep aggregator + builders; re-point the 3 broken streams (annual-report status, binder lifecycle, signature) to EntityAuditLog. **Remove `_DEDUP_ACTIONS` only after** the registry covers every category and timeline tests prove zero duplicate events (Phase 7) — not in the same step as the re-point. VatAuditLog + BinderIntakeEditLog were never read by timeline.

Child-entity client context resolves via `metadata_json.client_record_id` (§8) backed by the §8b expression index. Files changed: the five timeline files. Risk: Hebrew label parity; registry must guarantee single-source per category.

### 4a. Signature audit drawer — not timeline-only (correction #4)

The advisor signature detail route (`signature_request_routes_advisor.py:101`, `SignatureRequestWithAuditResponse`) embeds `audit_trail` (built in `signature_request_response_builder.py:build_with_audit` from `list_audit_events`) — consumed by the frontend **`SignatureRequestAuditDrawer`**. Deleting `SignatureAuditEvent` breaks this.

**Decision (single approach for this refactor) — keep the embedded `audit_trail` contract; re-source it from EntityAuditLog via the service layer.** The advisor detail endpoint `GET /signature-requests/{id}` and its `SignatureRequestWithAuditResponse` wrapper are **kept** (not deleted). **Layering (correction #5):** the signature service/route fetches audit items **through `AuditTrailService`** (e.g. a `get_entity_audit_items("signature_request", id)` that applies scope/redaction), and passes the **already-fetched items** to `build_with_audit(request, audit_items)`. The response builder must **not** query `EntityAuditLogRepository` (or any repository) directly — it only maps the items it is given. The frontend `SignatureRequestAuditDrawer` keeps consuming the embedded `audit_trail` and is **not** required to call the generic endpoint in this refactor.
- The generic `GET /audit/signature_request/{id}` **may** also serve signature audit (registry-gated), but it is **not** the drawer contract.
- `SignatureRequestWithAuditResponse` is **retained**. Only the per-item shape changes: the old `SignatureAuditEventResponse` (typed forensic columns) is replaced by an EntityAuditLog-derived item; **a replacement embedded item schema is defined** as part of Phase 6 (fields: `action`, `actor_type`, `actor_display_name`, `performed_at`, `note`, and the §8a forensic fields surfaced from `metadata_json`).
- **Preserve display of** `created/sent/viewed/signed/declined/canceled/expired`.
- Map `event_type → action` (`signature_request.<event>`), `actor_type` (`advisor→user`, `signer→external_signer`, `system→system`), and move `actor_name`, `ip_address`, `user_agent`, `content_hash`, `signed_document_key` into `metadata_json` **per action** (§8a).
- The drawer's backend contract shape is finalized **and** the frontend `SignatureRequestAuditDrawer` + generated types are regenerated/migrated **in Phase 6** (the embedded item shape changes there, so the frontend cannot lag). Phase 10 is only the final full contract-sync sweep, not the first signature-frontend migration.

## 5. Actor display snapshot (correction #5)

Renaming a user must **not** rewrite historical audit display. Joining `performed_by → users.full_name` at read time does exactly that.

- **EntityAuditLog gains `actor_display_name`** (nullable) — an **immutable snapshot** captured by `EntityAuditWriter` at write time from `current_user.full_name` (or signer name for external signers, or a system label). Read paths prefer the snapshot; the `performed_by` FK remains for filtering/joins but is not the display source.
- **UserAuditLog** already snapshots `email`; add `actor_display_name` + `target_display_name` for parity so admin-action history survives renames. (Low cost; recommended.)
- For `external_signer`/`system` rows `performed_by` is `NULL`; `actor_display_name` carries the signer name / system label so display never depends on a user join.

### 5a. Actor validation matrix (writer-enforced, fail-closed)

`EntityAuditWriter` validates the actor fields on every write; an invalid combination **fails and rolls back** the domain mutation (per §17).

| `actor_type` | `performed_by` | `actor_display_name` |
|---|---|---|
| `user` | **required** (FK users.id) | **required** |
| `system` | **must be NULL** | **required** (system label) |
| `external_signer` | **must be NULL** | **required** (signer name) |
| anything else | — | **fail validation** |

Rules: `actor_type` must be one of the three known values (unknown → reject). `user` without `performed_by`, or `system`/`external_signer` with a non-null `performed_by`, or any row missing `actor_display_name`, fails validation. No partial/invalid actor row is ever committed.

## 6. ENTITY_* constants — final

Canonical: `app/audit/audit_constants.py`. Rule: own `entity_type` only with an independent screen/route; child rows → parent entity_type + `metadata_json.<child>_id`. Every entity_type below must have an `AuditEntityRegistry` descriptor (§3a).

| Table/domain | entity_type | Granularity | Client resolver |
|---|---|---|---|
| client_records | `client` | row | self |
| businesses | `business` | row | `legal_entity → client_record` |
| legal_entities | `legal_entity` | row | via owning client/business |
| persons | `person` | row | via link → client (or firm-level) |
| person_legal_entity_links | `person_legal_entity_link` | row | via legal_entity → client |
| authority_contacts | `authority_contact` | row | `client_record_id` column |
| entity_notes | `note` | row | `entity_type/entity_id` → client |
| advance_payments | `advance_payment` | row | `client_record_id` column |
| charges | `charge` | row | `client_record_id` column |
| **invoices** | **`invoice`** | row | via `charge → client_record_id` (independent POST+GET API at `app/invoices/api/invoice_routes.py`; minimal `invoice.created`) |
| vat_work_items | `vat_work_item` | row | `client_record_id` column |
| vat_invoices | `vat_invoice` | row | via `work_item → client_record_id` |
| annual_reports | `annual_report` | row | `client_record_id` column |
| annual_report child tables | `annual_report` | **aggregate** | via parent report (rich child actions, §7a) |
| binders | `binder` | row | `client_record_id` column |
| binder_intakes | `binder_intake` | row | via `binder → client_record_id` |
| binder_intake_materials | `binder_intake` | aggregate | via intake → binder |
| binder_handovers | `binder_handover` | row | `client_record_id` column |
| permanent_documents | `document` | row | `client_record_id` column |
| signature_requests | `signature_request` | row (**sensitive**) | `client_record_id` column |
| tasks | `task` | row | `client_record_id` column |
| correspondence_entries | `correspondence` | row | `client_record_id` column |
| notifications | `notification` | row (partial) | `client_record_id` column |
| reminders | `reminder` | row (partial) | via source domain |
| tax_calendar_entries | `tax_calendar` | aggregate | firm-level |
| deadline_rules | `deadline_rule` (conditional) | row | firm-level |
| users | — (**no `ENTITY_USER`**) | — | auth/admin → UserAuditLog only |
| idempotency_keys / password_reset_tokens / junctions | — | none | infra |

Final `ENTITY_*` set = spec §9 list + `invoice` + `deadline_rule` (conditional); **no `ENTITY_USER`**. `ALLOWED_READ_ENTITY_TYPES` is derived from the registry, not hand-maintained separately.

## 7. ACTION_* constants — final (rich semantic)

Convention `domain.action`, constants only, **no raw strings in services**. Plain edits → `domain.updated` (diffs in old/new); lifecycle/evidence → specific verbs.

Core set (illustrative, **incomplete**): `client.*`, `business.*`, `legal_entity.updated`, `person.updated`, `person_legal_entity_link.created/deleted`, `advance_payment.created/updated/amount_changed/marked_paid/deleted`, `charge.issued/paid/canceled`, `invoice.created`, `vat_work_item.created/status_changed/assigned/filed/amount_overridden`, `vat_invoice.created/updated/amount_changed/deleted`, `annual_report.created/updated/status_changed/submitted/deleted`, `document.uploaded/replaced/approved/rejected/deleted`, `signature_request.created/sent/viewed/signed/declined/canceled/expired`, `binder.created/marked_full/reopened/marked_ready_for_handover/reverted_ready/handed_over/deleted/restored`, `binder_intake.received/updated`, `task.created/assigned/completed/canceled/deleted`, `correspondence.created/updated/deleted`, `notification.sent`, `reminder.created/canceled/failed`, `tax_calendar.generated`.

> **Binding action matrix is a Phase 0 deliverable — do not start implementation from this partial list.** Phase 0 produces a binding **mutation → entity_type → action → old/new payload → metadata_json → actor** matrix covering **every audited domain's create/update/delete flow**, explicitly including: authority_contacts, entity_notes, binder_handovers, **reminders fired events** (`reminder.fired`/`reminder.failed`), documents, notifications, and all annual-report child actions (§7a). The constant set above is finalized against that matrix before any writer is changed.

### 7a. Annual-report child actions — keep rich semantics (correction #8)

Do **not** collapse all child operations into `annual_report.updated`. Preserve:
- `annual_report.income_line_added/updated/deleted`
- `annual_report.expense_line_added/updated/deleted`
- `annual_report.annex_line_updated/deleted`
- `annual_report.schedule_completed`

`entity_type` stays `annual_report`; child identity goes in `metadata_json` (`section`, `line_id`/`schedule_id`, `line_number`). These replace the existing `ACTION_INCOME_*`, `ACTION_EXPENSE_*`, `ACTION_ANNEX_LINE_*` constants one-to-one (re-namespaced under `annual_report.*`). Plain detail edits with no dedicated verb use `annual_report.updated` + `metadata_json.section`.

## 8. metadata_json contract

`client_record_id` is **mandatory whenever a client context exists** (timeline/dashboard depend on it, indexed per §8b). The changed field itself goes in `old_value`/`new_value`, never metadata. Subject to the §16 sensitive-data policy.

| entity_type | required | optional |
|---|---|---|
| vat_invoice | client_record_id, vat_work_item_id, invoice_number, period, tax_year | business_id, source |
| vat_work_item | client_record_id, period, tax_year | source |
| advance_payment | client_record_id, period, tax_year | annual_report_id, source |
| annual_report (+children) | client_record_id, tax_year | section, line_id, schedule_id, line_number |
| document | client_record_id | business_id, annual_report_id, binder_id, document_type, version |
| signature_request | client_record_id, signer_name | per-action forensic (§8a) |
| binder / binder_intake | client_record_id, binder_id | binder_number, period_start/end, field_name |
| task | client_record_id | source_domain, source_id, assigned_to_user_id, assigned_role |
| charge / invoice | client_record_id | business_id, annual_report_id, invoice_id |

### 8a. Signature forensic metadata — per action (corrections #4, #9)

`signer_email` and `content_hash` are **currently nullable** in the domain. Do **not** make them hard audit requirements unless domain validation is explicitly planned. Capture-when-available, flag-when-missing:

| signature_request action | metadata captured | rule |
|---|---|---|
| created, sent | client_record_id, signer_name, signer_email (if present), business_id?, annual_report_id?, document_id? | advisor-initiated; no ip/user_agent |
| viewed, signed, declined | + ip_address, user_agent **when available** | signer-initiated; capture client forensics if the request carried them |
| signed | + content_hash **when available**, signed_document_key | if content_hash is missing, write `metadata_json.content_hash_missing = true` rather than failing the signature. **This refactor introduces no new queue/task workflow**; any remediation of missing hashes is a separately approved follow-up item, not part of this work |
| canceled, expired | client_record_id (+ reason for canceled) | no ip/user_agent required |

### 8b. Timeline performance — expression index (correction #7)

`metadata_json->>'client_record_id'` lookups must **not** be a JSONB scan. The Phase 1 migration adds a PostgreSQL **expression index**:
```
CREATE INDEX idx_entity_audit_client_ctx
  ON entity_audit_logs ((metadata_json->>'client_record_id'), performed_at);
```
`EntityAuditLogRepository.list_for_client_context` and the dashboard recent-activity query are written to use it (cast/compare as text consistently). SQLite dev path may table-scan; PostgreSQL is the performance target.

## 9. actor / user_id availability

Explicit service-level writes (no contextvars/after_flush/auto-audit). Actor flows from route `current_user`; writer snapshots `actor_display_name` (§5).

Missing-actor services to fix (add `actor_id` + display-name source from `current_user`): **advance_payment create/update, authority_contact add/update, permanent_document writer, legal_entity/person/link mutations**. Others already carry `created_by`/`performed_by`/`actor_id`. System events → `actor_type="system"`, `performed_by=None`; external signer → `actor_type="external_signer"`, `performed_by=None`, identity in metadata + `actor_display_name`.

## 10. Tables needing EntityAuditLog (from `3e2669e69e32_initial.py`, **42 tables**)

**Critical:** legal_entities, persons, person_legal_entity_links, advance_payments, vat_work_items, vat_invoices, permanent_documents, annual_reports + child tables (aggregate at parent), charges, invoices, signature_requests.
**High:** client_records, businesses, authority_contacts, entity_notes.
**Medium:** binders, binder_intakes, binder_intake_materials (aggregate), binder_handovers, tasks, correspondence_entries.
**Low/conditional:** notifications (user-initiated), reminders (significant events), tax_calendar_entries (batch = one event), deadline_rules (if UI-editable).
**No audit:** idempotency_keys, password_reset_tokens, junction tables, users (→ UserAuditLog).

Aggregate reminders: annual-report child ops use rich child actions (§7a) but stay at `entity_type=annual_report`; tax_calendar generation → one `tax_calendar.generated`; notification internal retry → none.

### 10b. BinderIntakeEditService preserved (correction #10)

**Do not delete `BinderIntakeEditService`.** It owns intake-edit business logic. Replace **only** its `BinderIntakeEditLogRepository` dependency with `EntityAuditWriter` (`binder_intake.updated`, field in `metadata_json`). Delete the log model + repository; keep the service and its `PATCH` route.

## 11. Old-model deletion checks

| Old model | EntityAuditLog preserves? | Actions | Remove | Risk |
|---|---|---|---|---|
| VatAuditLog | Yes | `vat_work_item.*`, `vat_invoice.*` | route, schema, repo, seed; rewrite test | low |
| AnnualReportStatusHistory | Yes | `annual_report.status_changed` (old/new=status) | route, schema, repo; rewrite test | low |
| BinderLifecycleLog | Yes (semantic + `metadata.field_name`) | `binder.*` | `/binders/{id}/audit`, schemas, repo, `BinderAuditService`, dashboard branch; rewrite test | medium (label/registry parity) |
| BinderIntakeEditLog | Yes | `binder_intake.updated` | model + repo (keep service §10b) | low |
| **SignatureAuditEvent** | Yes — verify | `signature_request.*` (evidence) | model, mixin, `SignatureAuditEventResponse` item shape; **keep `SignatureRequestWithAuditResponse`**, re-source its `audit_trail` from EntityAuditLog with a new embedded item schema (§4a); update timeline reader; rewrite test | **high — legal/forensic + embedded drawer**; per-action metadata §8a; backend response-shape + frontend drawer/types migrated in Phase 6 (final sync Phase 10) |

## 12. New repositories/services shape

`EntityAuditLogRepository`: keep `append`, `get_audit_trail`, `count_audit_trail`, `list_all_by_entities`, `list_recent`; **add** `list_by_entity`, `list_by_entities`, `list_for_client_context(client_record_id, entity_types=None, business_ids=None)` (uses §8b index), `list_recent_activity(limit)`. `append` gains `actor_type`, `actor_display_name`, `metadata_json`; old/new accept dict.

`EntityAuditWriter`: keep `record_create/update/delete/restore/status_change`; **add** `record_action(...)` and `record_external_action(...)` (actor_type external_signer, performed_by None). All capture `actor_display_name`. Append-only repository design per §17 (the audit repos must not expose mutation methods).

**Layering (correction #5):** `EntityAuditLogRepository`/`UserAuditLogRepository` do **DB access only** (append + read queries). `AuditTrailService` owns registry lookup, `resolve_scope`, authorization, redaction, and audit→response mapping; **all read consumers — routes, the signature response builder, timeline, dashboard — obtain audit items via `AuditTrailService`, never by calling the repository directly** for authorized/redacted reads. (Timeline/dashboard aggregation may use repository client-context queries internally, but a method that returns user-facing audit items applies the service's scope/redaction.)

Delete: `VatAuditLogRepository`, `AnnualReportStatusAuditRepository`, `BinderLifecycleLogRepository`, `BinderIntakeEditLogRepository`, `BinderAuditService`, `SignatureRequestAuditMixin`, and the deleted schemas. Keep canonical file names (do not rename to spec's example paths). **Model-registry / package-export cleanup** (`app/model_registry.py`, package `__init__` exports) removes references to the five deleted models.

## 13. Tests (provisional list — Phase 0 finalizes the change set)

43 test files live in the audit/timeline/dashboard/lifecycle area; the subset that directly asserts audit/history behavior is finalized in Phase 0. Currently identified:

`tests/audit/test_entity_audit_writer.py`, `test_audit_endpoint.py`, `test_audit_endpoint_filters.py`; `tests/core/test_openapi_audit_paths.py`; `tests/vat/api/test_vat_reports_audit.py`; `tests/annual_reports/api/test_annual_report_audit.py`, `tests/annual_reports/service/test_annual_report_generic_audit.py`, `test_financial_audit_snapshots.py`; `tests/binders/api/test_binder_audit.py`, `tests/binders/service/test_binder_lifecycle_service.py`, `test_binder_lifecycle_notification.py`; `tests/timeline/service/test_timeline_signature_lifecycle.py` + other `tests/timeline/*`; `tests/dashboard/service/test_recent_activity_service.py`, `tests/dashboard/api/test_dashboard_extended.py`; `tests/charges/service/test_billing_audit.py`; `tests/users/api/test_user_audit_logs.py`, `tests/users/services/test_audit_log_service.py`.

**New tests:** **JSON-object round-trip** — writer stores a dict/list and reader returns the same object (no `json.dumps`/`json.loads`), for `old_value`/`new_value`/`metadata_json` on EntityAuditLog **and** UserAuditLog.metadata_json; runtime response schema exposes objects, not strings; nullable `performed_by` + actor_type system/external_signer + `actor_display_name` snapshot survives user rename; per-domain writes → EntityAuditLog (VAT, annual-report status, annual-report child rich actions §7a, signature forensic §8a, binder lifecycle, binder_intake via preserved service §10b); generic route authorization (§3a, current model only): **401 unauthenticated; both ADVISOR and SECRETARY allowed**; 404 only when neither live entity nor usable history exists; soft-deleted readable-but-flagged; **hard-deleted authorized from audit metadata**; both roles receive the same allowed signature forensic fields. **No 403-by-role, owner/accountant, per-user ownership, or per-role-redaction tests** — those models do not exist. Signature advisor `audit_trail` parity (§4a); dashboard reads EntityAuditLog only; timeline single-source-per-category (no duplicates); UserAuditLog stays auth/admin-only; **append-only + atomicity tests (§17)**.

## 14. OpenAPI / frontend impact (plan only — do not regen)

| Route | Change | Regen? |
|---|---|---|
| `/audit/{type}/{id}` (`EntityAuditTrailResponse`) | Phase 1 changed JSON/actor fields; Phase 2 adds registry-backed reads, namespaced action values, and envelope-level `entity_deleted` | **yes — Phase 2 must export OpenAPI, run canonical `npm run gen:types`, update audit action labels/filters/formatter fallbacks plus narrow timeline label consumers, and run frontend checks. Never hand-edit `generated.ts`.** |
| `/vat/work-items/{id}/audit`, `/binders/{id}/audit`, `/annual-reports/{id}/audit` | **deleted** → generic | **yes** — frontend hooks switch endpoint **in-phase** (Phases 3/5/4), regen for that surface then; final sync Phase 10 |
| `/signature-requests/{id}` (`audit_trail`) | wrapper `SignatureRequestWithAuditResponse` **kept**; embedded item shape changes, re-sourced via `AuditTrailService` (§4a) | **yes** — drawer contract updated **in Phase 6**; final sync Phase 10 |
| `/dashboard/overview` (`recent_activity`) | source change | **verify** — confirm `activity_type`/label set unchanged after binder semantic actions; regen if item schema shifts |
| `/clients/{id}/timeline` | source change | **verify** — confirm `event_type` union unchanged; regen if it shifts |

## 15. Staged execution plan

**Each replacement phase (3–7) is end-to-end and self-contained: a legacy repo/schema/service is deleted only after *all* of its consumers — writer, reader, route/schema, AND every timeline/dashboard/drawer consumer — plus its tests are repointed in the same phase.** Legacy *tables* are never dropped in these phases; the single cleanup migration (Phase 9) drops them after all consumers are gone. This prevents a window where a reader/timeline/dashboard points at a deleted repo.

**Phase 0 — Baseline, exact inventory, enum audit, binding matrices.** Run baseline checks (verification list below). **Regenerate the exact inventory** (writer/reader/route/test counts, the 42-table list, per-table enum ownership-vs-sharing). **Produce the binding action matrix (§7/§9): mutation → entity_type → action → old/new payload → metadata_json → actor, covering every audited domain's create/update/delete plus authority_contacts, notes, binder_handovers, reminders fired events, documents, notifications, annual-report child actions.** **Enumerate `resolve_scope` (§3a) for every entity_type in §6.** **Produce the binding authorization matrix** (correction #4): `entity_type → allowed_roles → scope_policy → deleted_entity_policy → forensic/sensitive field visibility`, derived **only** from the existing role model (ADVISOR/SECRETARY) and existing active/deleted client filtering — no invented owner/accountant permissions. Snapshot `openapi.json`. Output → `docs/audit-refactor-phase-0-report.md`. Acceptance: all checks green; inventory + enum table + action matrix + scope-resolver list + authorization matrix produced. **No implementation starts until these matrices exist.**

**Phase 1 — Schema + JSON switch + actor threading + generic-audit contract sync (no legacy drops).** This phase is end-to-end for the surviving `entity_audit_logs`/`user_audit_logs` so existing writes never break and the response-shape change ships with its frontend.
- **Migration 1a** (§1b): JSONB-via-USING for `old_value`/`new_value` + `user_audit_logs.metadata_json`; add `metadata_json`, `actor_display_name`; add `actor_type` **nullable with temp `server_default="user"`**; `performed_by` nullable; portable `JSON().with_variant(JSONB,"postgresql")` model columns; indexes incl §8b expression index. Constants/registry scaffolding (§3a, §6, §7).
- **Serializer/reader switch** (§1b): drop `json.dumps` in `EntityAuditWriter._serialize_value` + `UserAuditLogRepository.create`; drop `json.loads` in `AuditLogService._to_dict`. `EntityAuditLogResponse` → `old_value`/`new_value`/`metadata_json` as JSON objects (+ `actor_type`, `actor_display_name`). `UserAuditLogResponse` → `metadata` as a JSON object (+ `actor_display_name`, `target_display_name`); **no `old_value`/`new_value`, no `actor_type`** (UserAuditLog has neither).
- **Actor threading — happens HERE, in Phase 1 (not Phase 2)** so migration 1b is safe: add minimal writer support (`actor_type` defaulting to `"user"`, `actor_display_name`) and **thread `current_user.full_name` + `actor_type` into all 30 existing EntityAuditLog write sites** (§5 of Phase 0 report). System/external-signer labels explicit where applicable. (The full `EntityAuditWriter` helpers + §5a validation matrix are Phase 2; Phase 1 only needs every existing write to pass `actor_type`.)
- **UserAuditLog snapshots — also Phase 1:** update the existing UserAuditLog writers (`user_auth_service.py`, `user_management_service.py`, password-reset) to capture `actor_display_name` (+ `target_display_name` where a target user exists). **No `actor_type` for UserAuditLog** — it has no such column; these are snapshot fields only.
- **Generic-audit frontend/OpenAPI sync — in Phase 1 (correction):** because `EntityAuditLogResponse.old_value/new_value` change **string → JSON object**, regenerate `openapi.json` + `generated.ts` and update `src/features/audit/` (contracts/hooks/`AuditTrailTable`/`EntityAuditTrailSection`) + any old/new-value **formatters** that assumed strings, then run the frontend checks. This is **not** deferred to Phase 2/10.
- **Migration 1b** (§1b): only after all existing writers pass `actor_type` — `actor_type NOT NULL` + drop the temp `server_default`.
- **Downgrade — fail-safe (Option A, §1b):** JSONB→Text casts; before re-imposing `performed_by NOT NULL` the downgrade **asserts no `performed_by IS NULL` rows and fails with a clear message otherwise** (it does **not** silently leave it nullable). 
- **Docs — `docs/domains/audit.md` (Phase 1):** create/update the canonical audit domain doc to reflect the Phase-1 reality — EntityAuditLog vs UserAuditLog shapes (JSON object `old_value`/`new_value`/`metadata_json` + `actor_type`/`actor_display_name` for EntityAuditLog; `metadata` object + `actor_display_name`/`target_display_name`, no `actor_type`, for UserAuditLog), the actor-snapshot convention, and the JSON-object response contract. Per the documentation-ownership rule, primary audit docs live here; touch `docs/backend/architecture.md` only if an architecture rule changes (e.g. the append-only audit-repository convention lands in Phase 2, not Phase 1).
- Acceptance: PostgreSQL up+down round-trip (1a+1b) clean and fail-safe; SQLite `create_all` works; existing audit writes pass `actor_type`/`actor_display_name` (EntityAuditLog) and snapshot fields (UserAuditLog); backend suite green; generic-audit OpenAPI/types regenerated and **frontend audit area green**; `docs/domains/audit.md` updated; legacy tables untouched.

**Phase 2 — Writer helpers / repository / registry / authz / validation.** Append-only audit repositories (§17 Option A); `EntityAuditWriter` (`record_action`/`record_external_action`, remove the actor-id-none no-op, enforce the **actor validation matrix §5a** and **fail-closed write validation §16**); `EntityAuditLogRepository` client-context queries; repository-backed `AuditEntityRegistry` + `resolve_scope` + current-role authorization (§3a). Exact normalized compact-JSON caps: old/new 32 KiB each, metadata 16 KiB. Populate the final §6 registry set, including `correspondence`; include conditional `deadline_rule` only if code confirms it is UI-editable. Enrich existing client/business/charge/annual-report writes with required `metadata_json`; namespace persisted actions. Add envelope-level `entity_deleted`; export OpenAPI, run canonical type generation, and update audit/timeline label consumers forced by namespacing without repointing timeline sources. Build/test the system-actor API but defer wiring signature auto-submit to Phase 6 because the surviving legacy annual-report status history still requires a user FK. Update `docs/domains/audit.md` and make the append-only audit repository rule explicit in `docs/backend/architecture.md`. Acceptance: 401 unauthenticated; both current roles allowed; 404 only with no live entity/usable history; soft/hard-deleted history readable and flagged; same allowed forensic visibility for both roles; actor validation, payload caps/allowlists, append-only, atomicity, metadata, OpenAPI/generated/frontend checks all pass.

**Each replacement phase that deletes or reshapes a route/schema also migrates that area's frontend consumer + regenerates the OpenAPI/`generated.ts` for that area + runs the frontend checks (correction #3).** The project decision is **no legacy wrappers / no backcompat / no temporary legacy routes** — so a deleted per-domain route cannot outlive its phase, and its frontend consumer moves in the **same** phase. Phase 10 is the **final full** verification + contract sync, **not** the first time any frontend consumer is migrated.

**Phase 3 — Replace VAT audit (end-to-end, incl. frontend).** Backend: repoint all `append_audit` writer sites → EntityAuditWriter; route audit reads through `AuditTrailService` (generic); rewrite VAT audit tests; delete `VatAuditLogRepository` + schema + per-domain route + seed builder. (Timeline/dashboard don't read VAT audit.) Frontend: switch the VAT audit hook/component to the generic `/audit/vat_work_item/{id}` endpoint; regenerate OpenAPI + `generated.ts` for this surface; run frontend checks for the VAT area. Table drop deferred to Phase 9. Acceptance: `/audit/vat_work_item/{id}` works; VAT backend suite + VAT frontend typecheck/tests green; no reference to `VatAuditLogRepository` or the old route remains.

**Phase 4 — Replace AnnualReportStatusHistory + child actions (end-to-end, incl. timeline + frontend).** Backend: status → `annual_report.status_changed`, child ops → rich actions (§7a); **repoint the timeline status-events reader (`TimelineRepository.list_annual_report_status_events`) in this same phase**; route reads through `AuditTrailService`; rewrite tests; delete `AnnualReportStatusAuditRepository` + schema + per-domain route. Frontend: switch the annual-report audit consumer to the generic endpoint; regen OpenAPI + types; run AR-area frontend checks. Table drop deferred. Acceptance: status + child history via generic route **and** timeline; AR backend + timeline + AR frontend checks green; no reference to the deleted repo/route remains.

**Phase 5 — Replace binder lifecycle + intake logs (end-to-end, incl. timeline + dashboard + frontend).** Backend: `_append_log` → `binder.*`; `BinderIntakeEditService` repointed to EntityAuditWriter (service preserved §10b); **repoint the timeline binder-lifecycle reader AND the dashboard `RecentActivityService` binder branch (drop `_serialize_binder`) in this same phase**; route reads through `AuditTrailService`; rewrite tests; delete `BinderAuditService`, `BinderLifecycleLogRepository`, `BinderIntakeEditLogRepository`, schemas, per-domain route. Frontend: switch the binder audit consumer to the generic endpoint; regen OpenAPI + types; run binder-area frontend checks. Tables dropped in Phase 9. Acceptance: timeline + dashboard + binder audit off EntityAuditLog; binder/timeline/dashboard backend + binder frontend checks green; no reference to deleted repos/route remains.

**Phase 6 — Replace SignatureAuditEvent (end-to-end, incl. timeline + embedded drawer + frontend).** Backend: `append_audit_event` → `signature_request.*` (external signer via record_external_action; per-action metadata §8a; content_hash-missing flag only §8a); **in the same phase repoint BOTH the timeline signature reader (`TimelineRepository.list_signature_lifecycle_events`) AND the embedded advisor `audit_trail` (signature service fetches items via `AuditTrailService`, passes to `build_with_audit`, §4a)**, keeping `SignatureRequestWithAuditResponse` with its **new embedded item schema**; rewrite backend response tests; delete `SignatureRequestAuditMixin` + `SignatureAuditEventResponse` item. Frontend: because the embedded `audit_trail` **item shape changes**, **update the `SignatureRequestAuditDrawer` contract in this same phase** — regen OpenAPI + `generated.ts` for the signature surface, adjust the drawer to the new item fields, run signature-area frontend checks. Acceptance: embedded `audit_trail` re-sourced via `AuditTrailService` with preserved `created/sent/viewed/signed/declined/canceled/expired` display; timeline signature events intact; signature backend + frontend drawer typecheck/tests green; OpenAPI documented + synced for this surface.

**Phase 7 — Rebuild timeline & dashboard, retire dedup.** With Phases 4–6 having already repointed each stream, apply the full event-source registry, finalize dashboard EntityAuditLog-only, add single-source tests; **remove `_DEDUP_ACTIONS` only once tests prove zero duplicates**. Acceptance: no duplicate events; timeline/dashboard suites green.

**Phase 8 — Add missing audit writes + actors.** Critical→high→medium→low (§10), driven by the Phase-0 binding matrix; add `actor_id`/display-name where missing (§9). Acceptance: each tier writes EntityAuditLog with required metadata; append-only/atomicity (§17) + actor-validation (§5a) tests pass per domain.

**Phase 9 — Cleanup migration + seeds.** Single migration drops the five now-consumerless legacy tables (downgrade recreates them with columns/indexes/FKs/constraints; drop only exclusively-owned enums, §1b). Update seed builders/orchestrator; **seed reset** runs clean. Model-registry/export cleanup. Acceptance: PostgreSQL up+down round-trip; seed reset green; suite green.

**Phase 10 — Final full verification & contract sync.** Frontend consumers were already migrated per-phase (3–6); this phase does the **final** full `openapi.json` + `generated.ts` regeneration (back up generated.ts first), confirms no drift remains anywhere, and runs the complete completion-criteria suite below across backend + frontend. Acceptance: contract sync clean repo-wide; all checks green.

### Completion criteria & verification commands
Backend (non-mutating): `pytest`, `vulture`, `pyright`, **`ruff format --check`** (not the mutating `ruff format`), `ruff check`, repo scripts (migration/role/pagination/unused/enums/schema), health checks, **PostgreSQL migration up+down round-trip**, **seed reset**, OpenAPI in sync.
Frontend (verification only): `lint`, `typecheck`, `test`, `format:check`, `arch:check`, `arch:check:strict`, `unused`. (Do **not** run `fix` as a verification step — it mutates.)
**Docs:** primary audit-behavior documentation lives in **`docs/domains/audit.md`** (canonical write/read flows, entity_type/action matrix, scope/authorization, sensitive-data policy); update affected domain/flow docs where behavior changes (VAT, annual-reports, binders, signature-requests, timeline, dashboard). Update **`docs/backend/architecture.md` only if an architecture rule actually changes** (e.g. the append-only audit-repository convention). **Model-registry / package-export cleanup** for the deleted models.

## 16. Sensitive-data policy — fail-closed writes, redacted reads

Audit payloads must never become an exfiltration surface. **Write-time filtering is fail-closed; redaction is a read-time concern.**

- **Forbidden in `old_value`/`new_value`/`metadata_json` (never logged):** `password_hash`, `token_hash`, `signing_token`, password-reset tokens, any secret/key, and **raw document content / file bytes**. Document audits store metadata only (storage_key, filename, mime, size, version) — not contents.
- **Write-time validation is fail-closed (not silent).** The writer validates each audit payload against a **per-action field allowlist**. An **unknown / non-allowlisted field fails validation and rolls back the domain mutation** (it is *not* silently dropped) — surfacing the mismatch immediately rather than leaking or losing data. Forbidden fields above are a hard reject.
- **PII handling:** `id_number` (ת.ז/ח.פ), `email`, `phone`, `ip_address`, `user_agent`, and document metadata are written **only where they are the legitimate audited field or required forensic context** (e.g. signature §8a), and only if on the allowlist. Where a full value isn't needed, the allowlist entry specifies the stored form (e.g. last-4 / hash of `id_number` when the change isn't to the id itself; truncated `user_agent`).
- **Payload size limits (fail-closed):** measure normalized compact JSON as UTF-8 bytes. `old_value` and `new_value` are capped at 32 KiB each; `metadata_json` is capped at 16 KiB. Oversized payloads fail validation and roll back. No summary-payload exception or silent truncation is introduced in Phase 2.
- **Read-time sensitive-data hook:** sensitive entity types (e.g. `signature_request`) pass through a service-owned hook after role/scope checks. Under the current model, both ADVISOR and SECRETARY preserve the same allowed forensic fields; there is no lower-privilege authenticated role and therefore no per-role redacted variant in this refactor. Forbidden data is rejected at write time and stored rows are never altered by reads.

## 17. Append-only & transactional integrity (correction #11)

- **Same transaction:** every audit write happens in the **same DB transaction** as the domain mutation it records (writer uses the request-scoped session, no separate commit). The route/service commits once.
- **Fail-closed:** if the audit insert fails, the **domain mutation rolls back** — a mutation must never succeed without its audit row. (Conversely, a rolled-back mutation leaves no orphan audit row.)
- **Append-only repository (BaseRepository reality).** `app/common/repositories/base_repository.py` `BaseRepository` **does expose** `update`, `delete`, `soft_delete`, and `hard_delete`. So "the methods don't exist" is false today and must be fixed deliberately. **Decision — Option A (preferred): audit repositories do not inherit `BaseRepository`.** `EntityAuditLogRepository` and `UserAuditLogRepository` are built on a small append-only base (or no base) exposing **append + read only** (`append`, `get_*`, `list_*`, `count_*`) and **no** `update`/`delete`/`hard_delete`. (Fallback Option B, only if non-inheritance proves impractical: inherit `BaseRepository` but **override `update`/`delete`/`soft_delete`/`hard_delete` to raise an explicit `AppError`/`NotImplementedError`.** Document whichever is chosen in Phase 2.) No audit mutation/delete **API** route exists either. Corrections are new rows.
- **Tests:** (a) **rollback test** — force the audit insert to raise, assert the domain row is absent after the request; (b) **append-only test** — assert the audit repositories expose no working mutation method (Option A: attribute absent; Option B: call raises); (c) **atomicity test** — domain failure leaves no audit row.

### Final acceptance (spec §13 + corrections)
Only `EntityAuditLog` + `UserAuditLog` remain; legacy tables dropped via the post-3–7 cleanup migration with reversible downgrade; JSONB (PostgreSQL) for old/new/metadata; actor display snapshots immutable to renames; generic route authorizes via `AuditEntityRegistry` and applies the sensitive-data visibility hook (same allowed forensic visibility for both current roles); signature drawer + timeline + dashboard work with no duplication; annual-report child operations keep rich semantic actions; audit writes are transactional + append-only; no raw action strings in services; all completion-criteria checks green.
