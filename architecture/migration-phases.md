# Migration Phases

Execution plan for moving the backend toward the target domain structure defined in
`target-domains-v1.md`, sequenced per the priorities in `domain-migration-map.md`.

## Conventions used by every phase

- **Structural moves only.** No method signatures, SQL, table names, enum values, or
  business rules change inside any phase. A phase is "move the file + repoint imports",
  never "rewrite the logic".
- **One phase = one focused PR.** Each phase is independently mergeable and independently
  revertable.
- Folder layout follows the existing convention: each domain package contains
  `api/`, `models/`, `repositories/`, `schemas/`, `services/` (api layer folder is named
  `api/`, not `routers/`).
- ORM models must stay importable before `configure_mappers()` runs. Any model file that
  moves must have its import line updated in `app/model_registry.py`.
- Any router that moves must have its import line updated in `app/router_registry.py`.
- Database table names are **not** renamed in any phase. The `legal_entities`, `persons`,
  `invoices`, `charges` tables already use target names; Python package renames do not touch them.

### Verified baseline facts (from codebase research)

- API layer folder is `api/` across all domains; routers are aggregated in
  `app/router_registry.py`; models are eagerly imported in `app/model_registry.py`.
- `app/invoice/` has **no `api/` folder** and is **not registered** in `router_registry.py`.
  Invoice is reachable only via `InvoiceService` / `InvoiceRepository`.
- There is **no client `Contact` model**. The only contact-like code is
  `authority_contact/` (separate domain), `clients/models/person.py` (legal-entity owners),
  and `businesses/services/business_contact_service.py` (derives contact info from the owner Person).
- `legal_entities`, `persons`, `person_legal_entity_links` models live under `clients/models/`
  and their repositories (`legal_entity_repository.py`, `person_repository.py`) under
  `clients/repositories/`.
- Alerts logic lives in `app/dashboard/services/dashboard_attention_service.py`.
- Actions is a **flat** package (`app/actions/*.py`), no `api/models/services` subfolders.
- Tests mirror app packages: `tests/<domain>/{api,repository,service}/`. There is no
  `tests/legal_entities/`, no `tests/contacts/`, no `tests/alerts/`. `tests/documents/`
  exists but is empty. `tests/invoice/api/` exists but is empty (no router to test).

---

## Phase 0 — Documentation (DONE)

Target structure (`target-domains-v1.md`), the domain-to-folder mapping
(`domain-migration-map.md`), and this phased plan (`migration-phases.md`) are written and
checked in. No code touched.

---

## Phase 1 — Legal Entities Extraction

### What
Extract the legal-entity / person identity models and their repositories out of `clients/`
into a new top-level package `app/legal_entities/`.

Moves:
- `app/clients/models/legal_entity.py` → `app/legal_entities/models/legal_entity.py`
- `app/clients/models/person.py` → `app/legal_entities/models/person.py`
- `app/clients/models/person_legal_entity_link.py` → `app/legal_entities/models/person_legal_entity_link.py`
- `app/clients/repositories/legal_entity_repository.py` → `app/legal_entities/repositories/legal_entity_repository.py`
- `app/clients/repositories/person_repository.py` → `app/legal_entities/repositories/person_repository.py`

### Why
The legal entity is the source of truth for all tax classifications and is consumed by far
more domains than `clients` (advance_payments, annual_reports, binders, notification, reports,
search, tax_calendar, timeline, vat_reports). It is currently buried inside `clients/models/`,
which misrepresents ownership. `target-domains-v1.md` lists `legal_entities/` as a first-class
target package.

### Files to move
- `app/clients/models/legal_entity.py`
- `app/clients/models/person.py`
- `app/clients/models/person_legal_entity_link.py`
- `app/clients/repositories/legal_entity_repository.py`
- `app/clients/repositories/person_repository.py`

### Files to update (imports)
Registries:
- `app/model_registry.py` (three model import lines: `legal_entity`, `person`, `person_legal_entity_link`)

Importers of `legal_entity` model / `LegalEntityRepository`:
- `app/advance_payments/repositories/advance_payment_aggregation_repository.py`
- `app/advance_payments/services/advance_payment_service.py`
- `app/annual_reports/repositories/report_repository.py`
- `app/annual_reports/services/base.py`
- `app/binders/repositories/binder_repository.py`
- `app/binders/services/binder_list_service.py`
- `app/clients/repositories/client_graph_writer.py`
- `app/clients/repositories/client_identity_repository.py`
- `app/clients/repositories/client_record_read_repository.py`
- `app/clients/repositories/client_record_repository.py`
- `app/clients/repositories/client_vat_stats_repository.py`
- `app/clients/services/client_onboarding_orchestrator.py`
- `app/clients/services/create_client_service.py`
- `app/notification/services/notification_context_resolver.py`
- `app/notification/services/notification_policy_service.py`
- `app/notification/services/notification_service.py`
- `app/reports/services/advance_payment_report.py`
- `app/reports/services/reports_service.py`
- `app/reports/services/vat_compliance_report.py`
- `app/search/services/search_service.py`
- `app/seed/builders/demo/advance_payments.py`
- `app/seed/validator.py`
- `app/tax_calendar/repositories/grouped_repository.py`
- `app/timeline/services/timeline_client_aggregator.py`
- `app/vat_reports/repositories/vat_work_item_query_repository.py`
- `app/vat_reports/services/client_context_service.py`
- `app/vat_reports/services/data_entry_invoices.py`
- `app/vat_reports/services/period_options.py`
- `app/vat_reports/services/vat_export_service.py`
- `app/vat_reports/services/vat_report_enrichment.py`

Importers of `person` / `person_legal_entity_link` / `PersonRepository` (additional to above):
- `app/businesses/services/business_contact_service.py`
- `app/clients/repositories/client_graph_writer.py`
- `app/clients/repositories/client_record_read_repository.py`
- `app/clients/repositories/client_record_repository.py`
- `app/clients/services/create_client_service.py`

Intra-package self-references inside the moved files (update to the new package path):
- moved `legal_entity_repository.py` imports `app.clients.models.legal_entity`
- moved `person_repository.py` imports `app.clients.models.person` and `...person_legal_entity_link`
- moved `legal_entity.py` has a `TYPE_CHECKING` import of `person_legal_entity_link`
- moved `person_legal_entity_link.py` references `Person`

Tests referencing legal-entity identity (update imports, do not rewrite assertions):
- `tests/conftest.py`
- `tests/helpers/identity.py`

### Non-goals
- Does **not** create a `legal_entities` service or router (none exists today; identity is
  managed via `clients` onboarding flow).
- Does **not** move `client_record.py` or any client lifecycle/ownership code.
- Does **not** rename the `legal_entities`, `persons`, or `person_legal_entity_links` tables.
- Does **not** change `LegalEntityRepository` / `PersonRepository` method signatures.
- Does **not** touch the cross-domain consumers' logic — import path only.

### Completion criteria
- `app/legal_entities/{models,repositories}/` contains the five moved files.
- `grep -r "app.clients.models.legal_entity\|app.clients.models.person\|app.clients.repositories.legal_entity_repository\|app.clients.repositories.person_repository" app tests` returns **zero** matches.
- `python -c "import app.model_registry"` succeeds (mappers configure).
- Full test suite passes unchanged (no test assertions modified).

---

## Phase 2 — Invoice API Layer

### What
Add the missing `api/` layer to `app/invoice/` and register it, so the invoice domain is
reachable over HTTP like every other domain. (Package stays `app/invoice/`; the rename to
`invoices/` is Phase 7, cosmetic.)

### Why
`domain-migration-map.md` marks invoices P1 specifically because the router layer is missing.
`InvoiceService.attach_invoice_to_charge` exists and is unit-tested
(`tests/invoice/service/test_invoice_service_rules.py`) but has no endpoint and is not in
`router_registry.py`. `tests/invoice/api/` exists but is empty, confirming the gap.

### Files to create
- `app/invoice/api/__init__.py`
- `app/invoice/api/invoices.py` — endpoints wrapping existing `InvoiceService`
  (attach invoice to charge, get charge invoice) using existing
  `InvoiceAttachRequest` / `InvoiceResponse` schemas. No new business logic; thin delegation.
- `app/invoice/api/routers.py` — aggregates the invoice router (mirrors `charge/api/routers.py`).
- `tests/invoice/api/test_invoices.py` — API-level tests for the new endpoints.

### Files to update
- `app/router_registry.py` — add `from app.invoice.api.routers import router as invoice_router`
  and `app.include_router(invoice_router, prefix="/api/v1")`.

### Non-goals
- Does **not** add new service methods or payment logic (Charge remains source of truth for
  payment per `target-domains-v1.md`).
- Does **not** rename the `invoice` package (Phase 7).
- Does **not** modify `InvoiceService`, `InvoiceRepository`, `Invoice` model, or schemas.

### Completion criteria
- `app/invoice/api/` exists and is registered in `router_registry.py`.
- New endpoints appear in the OpenAPI schema under `/api/v1`.
- `tests/invoice/api/test_invoices.py` passes; existing invoice service/repo tests still pass.
- No file outside `app/invoice/api/` and `app/router_registry.py` is modified.

---

## Phase 3 — Alerts Extraction

### What
Extract alert calculation/read logic from `dashboard/` into a new `app/alerts/` package.

Moves:
- `app/dashboard/services/dashboard_attention_service.py` →
  `app/alerts/services/alert_service.py` (file relocated; class/method names unchanged in this phase).

### Why
`target-domains-v1.md` defines Alerts as a distinct domain (internal signals from business
rules), explicitly separate from Dashboard (pure read model) and Notifications (outbound).
`domain-migration-map.md` marks it `extract` from `dashboard/`. The only consumer today is
`dashboard_overview_service.py`, so the blast radius is small.

### Files to move
- `app/dashboard/services/dashboard_attention_service.py`

### Files to update (imports)
- `app/dashboard/services/dashboard_overview_service.py` (the one current importer of
  `dashboard_attention_service`).

### Non-goals
- Does **not** add an alerts `api/` router (dashboard still surfaces alert data; no endpoint
  contract changes).
- Does **not** rename the service class or its methods, or alter calculation logic.
- Does **not** extract `recent_activity_service.py` or any other dashboard service.

### Completion criteria
- `app/alerts/services/alert_service.py` exists; `dashboard/services/dashboard_attention_service.py`
  is gone.
- `grep -r "dashboard_attention_service" app tests` returns zero matches.
- Dashboard endpoints return identical responses (existing `tests/dashboard/` suite passes unchanged).

---

## Phase 4 — Actions Vertical Slice

### What
Restructure the flat `app/actions/` package into the standard vertical-slice layout
(`app/actions/services/`), keeping module names so import surface is mechanical.

Moves (flat → `services/`):
- `app/actions/action_registry.py` → `app/actions/services/action_registry.py`
- `app/actions/action_helpers.py` → `app/actions/services/action_helpers.py`
- `app/actions/binder_actions.py` → `app/actions/services/binder_actions.py`
- `app/actions/business_actions.py` → `app/actions/services/business_actions.py`
- `app/actions/charge_actions.py` → `app/actions/services/charge_actions.py`
- `app/actions/obligation_orchestrator.py` → `app/actions/services/obligation_orchestrator.py`
- `app/actions/report_deadline_actions.py` → `app/actions/services/report_deadline_actions.py`
- `app/actions/vat_report_actions.py` → `app/actions/services/vat_report_actions.py`

### Why
`domain-migration-map.md` marks actions `restructure: flat → vertical slice`.
`target-domains-v1.md` defines Actions as an application layer that centralizes command
definitions per domain. The flat layout is inconsistent with every other package.

### Files to update (imports)
External consumers of `app.actions.*`:
- `app/clients/services/client_update_service.py` (`obligation_orchestrator`)
- `app/clients/services/client_onboarding_orchestrator.py` (`obligation_orchestrator`,
  including two function-local imports of `_years_to_generate`)
- `app/clients/services/impact_preview_service.py` (`obligation_orchestrator`)
- `app/vat_reports/api/serializers.py` (`vat_report_actions`)
- `app/businesses/services/client_business_service.py` (`action_registry`)
- `app/charge/services/charge_response_builder.py` (`charge_actions`)
- `app/charge/services/charge_query_service.py` (`charge_actions`)
- `app/binders/services/binder_list_service.py` (`action_registry`)
- `app/annual_reports/services/base.py` (function-local import of `report_deadline_actions`)

Internal cross-imports inside `action_registry.py` (points at sibling action modules:
`binder_actions`, `business_actions`, `charge_actions`, `report_deadline_actions`,
`vat_report_actions`).

Tests:
- `tests/actions/*` import paths (e.g. `tests/actions/test_business_actions.py`).

### Non-goals
- Does **not** add `actions/api/`, `actions/models/`, or `actions/schemas/` (no action
  registry/execution endpoints exist yet; those are future feature work, not this move).
- Does **not** rename or merge any action module or change action definitions.

### Completion criteria
- All eight modules live under `app/actions/services/`; `app/actions/` top level contains only
  `services/` (plus `__init__.py`).
- `grep -rE "from app.actions.(action_registry|action_helpers|binder_actions|business_actions|charge_actions|obligation_orchestrator|report_deadline_actions|vat_report_actions)" app tests` returns zero matches (all now go through `app.actions.services.*`).
- `tests/actions/` passes; consumers in clients/charge/binders/vat/businesses/annual_reports still pass.

---

## Phase 5 — Documents Parent Domain

### What
Introduce `app/documents/` as the parent domain and nest the existing
`permanent_documents/` package under it.

Moves:
- `app/permanent_documents/` → `app/documents/permanent_documents/` (entire package:
  `api/`, `models/`, `repositories/`, `schemas/`, `services/`).

### Why
`target-domains-v1.md` defines Documents as the parent domain that "replaces
permanent_documents as parent domain", with `permanent_documents/` as a child.
`domain-migration-map.md` marks it `create parent + move`.

### Files to move
Whole `app/permanent_documents/` tree, including:
- `app/permanent_documents/api/{permanent_documents.py,permanent_document_actions.py,routers.py}`
- `app/permanent_documents/models/permanent_document.py`
- `app/permanent_documents/repositories/{permanent_document_repository.py,permanent_document_query_repository.py}`
- `app/permanent_documents/schemas/permanent_document.py`
- `app/permanent_documents/services/*` (service, action service, response_builder, constants,
  messages, upload_constraints)

### Files to update (imports)
- `app/model_registry.py` — `import app.permanent_documents.models.permanent_document` →
  `app.documents.permanent_documents.models.permanent_document`.
- `app/router_registry.py` — `from app.permanent_documents.api.routers import router` →
  `app.documents.permanent_documents.api.routers`.
- Every external consumer of `app.permanent_documents.*` — enumerate immediately before the PR
  with `grep -rln "app.permanent_documents" app tests` and repoint each (known consumers include
  `timeline/`, `reports/`, `search/`, `seed/`, plus intra-package self-imports).
- `tests/permanent_documents/*` import paths.

### Non-goals
- Does **not** create generic `documents/` models/services/router yet (no generic Document
  entity exists today); this phase only establishes the parent folder and relocates the child.
- Does **not** merge `permanent_documents` logic into a generic document service.
- Does **not** change the permanent-document table name or API routes.

### Completion criteria
- `app/documents/permanent_documents/` contains the full moved tree; old
  `app/permanent_documents/` is gone.
- `grep -r "app.permanent_documents" app tests` returns zero matches.
- `model_registry` imports cleanly; permanent-document routes unchanged in OpenAPI.
- `tests/permanent_documents/` passes with updated import paths only.

---

## Phase 6 — Contacts Extraction

### What
Scaffold the `app/contacts/` package (`api/`, `models/`, `repositories/`, `schemas/`,
`services/` with `__init__.py` placeholders).

### Why
`target-domains-v1.md` defines a Contacts domain (client people contacts, distinct from
Authority Contacts). `domain-migration-map.md` marks it P1 `create/extract` with the caveat
"check if any code exists in clients/".

**Research result:** there is **no client `Contact` model anywhere**. The only contact-like
code is `authority_contact/` (a separate target domain), `clients/models/person.py`
(legal-entity owners, moved in Phase 1), and `businesses/services/business_contact_service.py`
(derives email/phone from the owner Person). There is nothing to *extract* — a real client
Contact entity is **net-new feature work**, which would introduce business logic and therefore
violates the "structural moves only" rule of this plan.

This phase is therefore limited to creating the empty package skeleton so the target structure
exists. Actual Contact modeling, schemas, repository, and endpoints are tracked as a separate
feature effort, **out of scope** for this structural migration.

### Files to create
- `app/contacts/__init__.py` and empty `api/`, `models/`, `repositories/`, `schemas/`,
  `services/` packages (each with `__init__.py`).
- `tests/contacts/__init__.py` (mirrors layout; no test cases yet).

### Files to update
- None. No existing import is repointed, because no contact code is moving.

### Non-goals
- Does **not** define a `Contact` model, schema, repository, service, or router (that is
  feature work with business logic, explicitly excluded here).
- Does **not** touch `authority_contact/`, `person.py`, or `business_contact_service.py`.
- Does **not** register anything in `model_registry.py` or `router_registry.py` (nothing to register).

### Completion criteria
- `app/contacts/` skeleton exists and imports cleanly.
- No behavior change anywhere; full test suite passes unchanged.
- A follow-up ticket exists describing the net-new Contact entity (out of this plan's scope).

---

## Phase 7 — Cosmetic Renames

Pure package renames (Python path only; table names untouched). Each rename is its own PR.
Ordered by **least import surface first** to keep early PRs trivial and de-risk the large ones.

### Rename list (one at a time, not all at once)

1. `app/correspondence/` → `app/communications/`
2. `app/authority_contact/` → `app/authority_contacts/`
3. `app/notification/` → `app/notifications/`
4. `app/charge/` → `app/charges/`
5. `app/invoice/` → `app/invoices/`  *(includes the `api/` layer added in Phase 2)*
6. `app/vat_reports/` → `app/vat/`

### Order (by least import surface first)

Measured via `grep -rln "app\.<pkg>" app` (file count referencing each package):

| Order | Rename | Importing files |
|---|---|---|
| 1 | correspondence → communications | 8 |
| 2 | authority_contact → authority_contacts | 11 |
| 3 | notification → notifications | 17 |
| 4 | charge → charges | 27 |
| 5 | invoice → invoices | ~5 (model_registry, seed/charges, timeline_service, plus Phase 2 api) |
| 6 | vat_reports → vat | 75 |

`invoice` is placed at step 5 (after charge) so it carries the `api/` layer added in Phase 2;
its raw import surface is the smallest, but sequencing it after `charge` avoids churning the
charge↔invoice FK references twice. `vat_reports` is deliberately last — 75 referencing files
make it the highest-risk rename and it should land alone, after the mechanics are proven on
smaller packages.

Each rename PR updates, for its package:
- the package directory name,
- intra-package self-imports,
- the corresponding line(s) in `app/model_registry.py` and `app/router_registry.py`,
- every `app.<oldpkg>` reference across `app/` and `tests/`,
- the mirrored `tests/<oldpkg>/` directory name.

### Completion criteria (per rename)
- `grep -r "app\.<oldpkg>" app tests` returns zero matches.
- `tests/<newpkg>/` mirrors the renamed package and passes.
- `model_registry` / `router_registry` import cleanly; OpenAPI routes and DB table names unchanged.
- Each rename merges and the suite is green before the next rename starts.

---

## Phase 8 — Deferred Domains

Domains marked P3 / `defer` in `domain-migration-map.md`. Do **not** execute speculatively;
each has an explicit trigger.

### national_insurance
- **Current state:** lives inside `app/annual_reports/services/ni_engine.py` (plus
  `tax_engine.py` consumers within annual_reports).
- **Target:** `app/national_insurance/`.
- **Trigger condition:** extract only when NI gets its own work items / lifecycle / endpoints
  (the `create_ni_record` … `get_ni_balance_summary` API surface in `target-domains-v1.md`
  section 17) — i.e. when NI stops being a calculation helper for annual reports and becomes a
  domain with its own persisted records. Until then `ni_engine.py` stays put.

### permissions
- **Current state:** no `permissions/` package; role checks (ADVISOR/SECRETARY) are inlined in
  service/guard code (e.g. `clients/guards/`).
- **Target:** `app/permissions/` (or stay inside `users/core` if it remains thin).
- **Trigger condition:** extract only when permission logic is duplicated across ≥3 domains or a
  non-role-based access rule appears. Per `target-domains-v1.md` section 29 the model is
  role-only with no per-entity ACL, so a thin inline implementation is acceptable indefinitely.

### imports (client import)
- **Current state:** no `imports/` package. (`app/seed/` is demo seeding, **not** the product
  client-import feature.)
- **Target:** `app/imports/` (client import only; export already lives in `reports/`).
- **Trigger condition:** create only when the client bulk-import feature
  (`validate_import` / `preview_import` / `import_clients` / `commit_import` from
  `target-domains-v1.md` section 30) is actually built. This is net-new feature work, not a move.

### Non-goals for Phase 8
- No package is created until its trigger fires.
- No business logic is written as part of "migration" — when a trigger fires, that becomes a
  feature effort with its own plan, not a structural move under this document.
