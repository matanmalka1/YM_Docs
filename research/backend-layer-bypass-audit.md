## Scope

This file owns only:

- A point-in-time audit of backend paths that bypass or invert the required
  `Router -> Service -> Repository -> DB` dependency flow.
- Evidence, architectural impact, and remediation boundaries for each finding.

This file must not contain:

- A replacement for `docs/backend/architecture.md`.
- New backend architecture decisions.
- A claim that every non-standard module location is a runtime defect.

Source of truth: research/audit only

# Backend Layer Bypass Audit

Audit date: 2026-07-27.

Canonical rules:

- `docs/adr/0002-router-service-repository.md` requires
  `Router -> Service -> Repository -> DB`.
- `docs/backend/architecture.md` requires routers to call services, services to own business
  logic and DTO mapping, and repositories to own database access only.
- Routed domains must use the `api/`, `services/`, `repositories/`, `schemas/`, and `models/`
  vertical-slice structure.

## Method and classification

The audit checked:

1. Repository imports and repository construction under every backend `api/` directory.
2. Transitive paths where an API module calls a response builder, serializer, or root-level
   helper that then accesses a repository.
3. Direct SQLAlchemy query execution from services.
4. Reverse dependencies from repositories to services.
5. Persistence types imported into schemas.
6. Cross-cutting authentication dependencies that access persistence from the API boundary.
7. Repository imports of domain exceptions, transport schemas, other domain repositories, and
   services.
8. `commit()`, `rollback()`, and transaction ownership outside services.
9. FastAPI exceptions raised from services or repositories.
10. Endpoint control flow, multi-service composition, repeated service calls, and domain
    not-found/validation decisions.
11. SQLAlchemy/session access and repository construction in root-level domain helpers,
    background entrypoints, infrastructure dependencies, response builders, and aggregators.

Classifications:

- **Direct bypass** — an API handler constructs or calls a repository.
- **Transitive bypass** — an API handler calls a non-service helper that accesses a repository.
- **Reverse dependency** — a repository imports or calls a service.
- **Misplaced service** — service behavior exists outside the domain's `services/` directory.
- **Persistence coupling** — a schema or transport type depends on a repository-owned type.
- **Infrastructure exception candidate** — cross-cutting API infrastructure accesses a
  repository directly and requires an explicit architectural decision.

Severity:

- **High** — an active request path bypasses the service boundary or creates an inverted
  dependency.
- **Medium** — behavior follows the logical layers but violates structural ownership, or a
  transport type depends on persistence.
- **Low** — a cross-cutting boundary is ambiguous and should be made explicit.

## Executive inventory

| ID | Finding | Classification | Severity |
|---|---|---|---|
| BL-001 | Advance-payment routers query turnover repositories directly | Direct bypass | High |
| BL-002 | Bulk-generation preview queries its repository directly | Direct bypass | High |
| BL-003 | Invoice route performs repository lookup and not-found decision | Direct bypass | High |
| BL-004 | Binder handover route queries handover membership directly | Direct bypass | High |
| BL-005 | Signature-request client route queries client records directly | Direct bypass | High |
| BL-006 | Signature response builder hides repository access behind API response assembly | Transitive bypass | High |
| BL-007 | Permanent-document response builder performs database enrichment | Transitive bypass | High |
| BL-008 | Report services are outside the required `services/` layer | Misplaced service | Medium |
| BL-009 | Client graph repository imports a service and owns orchestration | Reverse dependency | High |
| BL-010 | Advance-payment schema imports a repository-owned projection | Persistence coupling | Medium |
| BL-011 | Authentication dependency queries the user repository directly | Infrastructure exception candidate | Low |
| BL-012 | Grouped VAT routes call a root helper that queries repositories | Transitive bypass | High |
| BL-013 | Background jobs bypass service-owned transaction boundaries | Direct bypass | High |
| BL-014 | Idempotency API dependency owns repository and transaction behavior | Infrastructure exception candidate | High |
| BL-015 | Work-queue service is split across repository-backed root/item helpers | Misplaced service | Medium |
| BL-016 | VAT service behavior is split across repository-backed root helpers | Misplaced service | Medium |
| BL-017 | Timeline and notification query services are partly outside `services/` | Misplaced service | Medium |
| BL-018 | Domain guards and lookup helpers perform repository access outside services | Misplaced service | Medium |
| BL-019 | Reports export service raises FastAPI transport exceptions | Layer responsibility violation | Medium |
| BL-020 | Client repository raises a domain not-found error | Layer responsibility violation | Medium |
| BL-021 | Advance-payment repository orchestrates a VAT repository | Cross-domain repository orchestration | High |
| BL-022 | Search repository imports a transport-schema enum | Persistence coupling | Medium |
| BL-023 | Routers orchestrate multiple services for one endpoint operation | Layer responsibility violation | High |
| BL-024 | Routers own domain not-found, validation, and transition branching | Layer responsibility violation | High |

## Detailed findings

### BL-001 — advance-payment routers query turnover repositories directly

Evidence:

- `backend/app/advance_payments/api/advance_payment_routes.py:90-94` constructs
  `TurnoverLookupRepository` and resolves turnover for list rows.
- `backend/app/advance_payments/api/advance_payment_routes.py:198-201` calls the service for the
  payment and then calls `TurnoverLookupRepository` for additional response data.
- `backend/app/advance_payments/api/advance_payment_routes.py:50-65` contains derived response
  decisions for available turnover, VAT mismatch, and missing turnover.

Impact: the route owns repository access, business branching, derived state, and DTO enrichment.
Callers of `AdvancePaymentService` do not receive the same complete representation unless they
reproduce the router logic.

Required boundary: a read or analytics service should resolve turnover and return an
`AdvancePaymentRow` or a typed service result. The router should call one service method and return
the mapped response.

### BL-002 — bulk-generation preview queries its repository directly

Evidence:

- `backend/app/advance_payments/api/advance_payment_routes_overview.py:210-219` constructs
  `AdvancePaymentGenerationRepository`, runs two queries, and maps the ineligible-client reason.

Impact: repository access and response mapping are owned by the transport layer. The literal
`frequency_not_set` decision also becomes route-owned behavior.

Required boundary: add a preview method to the generation or advance-payment service that returns
the eligible count and typed ineligible-client results.

### BL-003 — invoice route performs repository lookup and not-found decision

Evidence:

- `backend/app/invoices/api/invoice_routes.py:55-57` calls
  `InvoiceRepository.get_by_charge_id()` and raises `NotFoundError`.

Impact: lookup semantics and the domain not-found decision bypass `InvoiceService`.

Required boundary: expose `InvoiceService.get_by_charge_id()` and keep repository lookup and
`INVOICE_NOT_FOUND` handling inside the service.

### BL-004 — binder handover route queries handover membership directly

Evidence:

- `backend/app/binders/api/binder_routes_receive_return.py:101-112` creates the handover through
  `BinderHandoverService`, then queries `BinderHandoverRepository` directly.
- `backend/app/binders/api/binder_routes_receive_return.py:113-123` manually assembles the response.

Impact: one endpoint operation is split between service orchestration and router persistence
access. The service result is incomplete for its API consumer.

Required boundary: the handover service should return a typed result containing the handover and
binder IDs, or return the complete response schema.

### BL-005 — signature-request client route queries client records directly

Evidence:

- `backend/app/signature_requests/api/signature_request_routes_client.py:67-79` calls the
  signature service, then uses `ClientRecordRepository.list_by_ids()` to build an office-number
  map.
- `backend/app/signature_requests/api/signature_request_routes_client.py:80-89` performs DTO
  enrichment and pagination response assembly.

Impact: service-owned enrichment and DTO mapping are performed in the router. The same domain also
has a separate response builder, so response construction is duplicated.

Required boundary: return enriched list results from a signature-request read service and use one
canonical response-mapping path.

### BL-006 — signature response builder hides repository access

Evidence:

- `backend/app/signature_requests/signature_request_response_builder.py:14-17` constructs
  `BusinessRepository` and `ClientRecordRepository`.
- `backend/app/signature_requests/signature_request_response_builder.py:70-81` queries both
  repositories during response enrichment.
- API call sites include:
  - `backend/app/signature_requests/api/signature_request_routes_client.py:50`
  - `backend/app/signature_requests/api/signature_request_routes_advisor.py:64`
  - `backend/app/signature_requests/api/signature_request_routes_advisor.py:91`
  - `backend/app/signature_requests/api/signature_request_routes_advisor.py:108`

Impact: the route appears to perform response mapping only, but the builder crosses the database
boundary. The builder is effectively a read service outside `services/`.

Required boundary: move repository-backed enrichment into a signature-request read service.
Pure response serializers may remain under `api/` only when they do not access the database.

### BL-007 — permanent-document response builder performs database enrichment

Evidence:

- `backend/app/documents/permanent_documents/permanent_document_response_builder.py:15-23`
  queries client records and maps Pydantic responses.
- It is called repeatedly from:
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:70`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:101`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:126`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:204`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:216`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes.py:240`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes_actions.py:43`
  - `backend/app/documents/permanent_documents/api/permanent_document_routes_actions.py:62`

Impact: eight active route paths perform persistence-backed enrichment after their service call.
This spreads one endpoint operation across service and API-adjacent database access.

Required boundary: a permanent-document read/response service should return enriched response
schemas. A remaining response builder must be a pure mapper without a DB session.

### BL-008 — report services are outside the required services layer

Evidence:

- `backend/app/reports/api/report_routes.py:40`, `:50`, and `:81` instantiate report service
  classes.
- The classes live at:
  - `backend/app/reports/vat_compliance_report.py:16`
  - `backend/app/reports/advance_payment_report.py:11`
  - `backend/app/reports/annual_report_status_report.py:12`
- These classes construct and call repositories.

Impact: runtime flow is logically API-to-service-to-repository, but file ownership violates the
mandatory vertical slice. Static dependency checks cannot reliably distinguish these services
from root-level helpers.

Required boundary: move the classes into `backend/app/reports/services/` and rename files using
the required domain prefix and `_service.py` suffix. Update all callers, including
`report_reports_export_service.py`.

### BL-009 — client graph repository imports a service and owns orchestration

Evidence:

- `backend/app/clients/repositories/client_graph_writer.py:4` imports
  `get_client_or_raise` from `clients.services.client_service`.
- `backend/app/clients/repositories/client_graph_writer.py:34-56` loads multiple domain objects,
  decides field ownership, mutates `Person`, `LegalEntity`, and `ClientRecord`, flushes, and
  assembles a full-record dictionary.
- `backend/app/clients/services/client_update_service.py:169-170` calls this repository helper.

Effective dependency:

```text
ClientUpdateService
  -> client_graph_writer repository
    -> client_service
      -> ClientRecordRepository
```

Impact: the repository depends upward on a service and owns cross-model orchestration and business
field routing. This creates a cycle-prone boundary and prevents repositories from remaining
database-only.

Required boundary: keep graph-update orchestration and not-found decisions in
`ClientUpdateService`. Repositories should expose focused persistence operations for each owned
model and return ORM objects or typed projections only.

### BL-010 — advance-payment schema imports a repository-owned projection

Evidence:

- `backend/app/advance_payments/schemas/advance_payment.py:21-23` imports
  `TurnoverResolution` from `advance_payment_turnover_lookup_repository`.
- `AvailableTurnover.from_resolution()` and `VatTurnoverMismatch.from_comparison()` accept that
  repository-owned type.

Impact: the public transport schema depends on persistence-layer ownership. Moving or replacing
the repository requires modifying the API contract module, and the dependency direction is
opposite to the intended layering.

Required boundary: move `TurnoverResolution` to a neutral typed projection module owned by the
domain, or have the service map it to schema fields before schema construction.

### BL-011 — authentication dependency queries the user repository directly

Evidence:

- `backend/app/users/api/user_deps.py:46-47` constructs `UserRepository` and loads an auth subject.
- `backend/app/users/api/user_deps.py:49-59` applies active-user and token-version decisions.

Context: `docs/architecture/security.md` explicitly requires runtime authentication to be enforced
by `get_current_user`, and requires route signatures to use `CurrentUser`. It does not explicitly
authorize the dependency to bypass services.

Impact: all protected endpoints transitively depend on a repository through the API dependency.
This may be an intentional cross-cutting infrastructure boundary, but the exception is not
documented in the backend architecture.

Required boundary: either:

1. delegate active-user and token-version validation to a user authentication service, leaving the
   dependency responsible for bearer parsing and transport errors; or
2. document a narrow authentication-dependency exception in the owning architecture document.

Do not change this path without preserving the mandatory runtime checks from
`docs/architecture/security.md`.

### BL-012 — grouped VAT routes call a root helper that queries repositories

Evidence:

- `backend/app/vat/api/vat_routes_grouped.py:34-41` calls
  `vat_grouped_enrichment.get_groups()` directly.
- `backend/app/vat/api/vat_routes_grouped.py:60-69` calls
  `vat_grouped_enrichment.get_group_items_enriched()` directly.
- `backend/app/vat/vat_grouped_enrichment.py:25-34` executes grouped repository queries.
- `backend/app/vat/vat_grouped_enrichment.py:52-77` queries, enriches, and serializes the item
  list.
- `backend/app/vat/vat_grouped_enrichment.py:80-96` constructs `ClientRecordRepository` and
  implements client-name filtering decisions.

Impact: both endpoints completely skip a VAT service. The root helper owns query selection,
filtering, enrichment, invalid-group handling, and response serialization.

Required boundary: move grouped VAT query and enrichment behavior into a dedicated
`vat_grouped_service.py`. The router should instantiate that service and call one method per
endpoint.

### BL-013 — background jobs bypass service-owned transaction boundaries

Evidence:

- `backend/app/core/background_jobs.py:20-29` constructs
  `SignatureRequestRepository`, passes it into `expire_overdue_requests()`, and calls
  `db.commit()`/`db.rollback()`.
- `backend/app/core/background_jobs.py:97-100` repeats direct repository construction for the
  recurring job.
- `backend/app/core/background_jobs.py:83-94` owns commit and rollback for every scheduled task.
- `backend/app/core/background_jobs.py:35-53` and `:56-80` also commit or roll back around service
  calls, so transaction ownership remains in the entrypoint even where repository construction is
  not direct.

Impact: the background entrypoint bypasses normal service construction and owns transaction
boundaries, despite the backend rule that only services call `commit()`, `rollback()`, or
`transaction()`. Passing a repository into a service function also exposes persistence details at
the entrypoint.

Required boundary: expose a signature-request expiry service method that constructs its own
repository and owns the transaction. Background jobs should open/close the session and call one
service method; transaction policy should remain inside the service.

### BL-014 — idempotency API dependency owns repository and transaction behavior

Evidence:

- `backend/app/infrastructure/idempotency/dependency.py:60-67` executes request-level behavior and
  constructs `IdempotencyKeyRepository`.
- `backend/app/infrastructure/idempotency/dependency.py:69-102` reserves and replays keys while
  handling persistence conflicts.
- `backend/app/infrastructure/idempotency/dependency.py:104-119` releases or completes repository
  state and can call `db.rollback()`.
- `docs/architecture/security.md` recommends `IdempotencyService` for storing fingerprints and
  replaying cached responses, but no such service currently owns this behavior.

Impact: every idempotent API endpoint receives a dependency object that combines transport
handling, business execution, persistence, transaction control, and FastAPI response creation.
This is a cross-cutting `API/infrastructure -> Repository -> DB` bypass.

Required boundary: separate a persistence-aware idempotency service from the FastAPI dependency.
The dependency should parse headers and request metadata; the service should reserve, compare,
release, complete, and apply transaction rules. Because this wraps domain service execution, the
final transaction design should be explicit rather than changed mechanically.

### BL-015 — work-queue service is split across repository-backed root/item helpers

Evidence:

- `backend/app/work_queue/services/work_queue_service.py:13-22` imports item builders from
  `work_queue/items/`.
- `backend/app/work_queue/items/billing_items.py:26` and `:54` query advance-payment and charge
  repositories.
- `backend/app/work_queue/items/binder_items.py:16` queries the binder repository.
- `backend/app/work_queue/items/tax_items.py:33` and `:60` query VAT and annual-report
  repositories.
- `backend/app/work_queue/items/common.py:109` queries client identities.
- `backend/app/work_queue/services/work_queue_service.py:36` imports
  `work_queue_source_lookup`.
- `backend/app/work_queue/work_queue_source_lookup.py:64-172` queries five repositories and maps
  business status into `SourceState`.

Impact: `WorkQueueService` is nominally the service boundary, but much of its database access and
derived-state logic lives in folders and root modules outside `services/` and `repositories/`.
These helpers are service implementations with direct repository dependencies.

Required boundary: keep pure item mappers separate, but move repository-backed orchestration into
explicit work-queue query services or specialized work-queue read repositories. Pure builders
must receive already-loaded typed inputs.

### BL-016 — VAT service behavior is split across repository-backed root helpers

Evidence:

- `backend/app/vat/services/vat_report_service.py:12-15` imports root-level VAT query/enrichment
  helpers.
- `backend/app/vat/services/vat_report_service.py:125-147` delegates enriched reads to
  `vat_report_enrichment`.
- `backend/app/vat/vat_report_enrichment.py:27-31` directly constructs client and legal-entity
  repositories; the same module also imports user, invoice, and work-item repositories.
- `backend/app/vat/vat_period_options.py:31-40` directly loads client, legal entity, and VAT work
  items while computing available periods.
- `backend/app/vat/vat_report_queries.py:44-163` accepts repositories and combines queries with
  not-found decisions and response construction.
- `backend/app/vat/vat_data_entry_common.py:51-60` accepts two repositories and coordinates an
  aggregate read followed by a work-item write.
- `backend/app/vat/vat_data_entry_common.py:102-128` combines repository reads with tax-limit
  business rules and domain errors.
- `backend/app/vat/vat_work_item_metadata.py:20-77` performs locked reads, mutation validation,
  work-item writes, and audit orchestration outside `services/`.

Impact: the public service delegates service-owned database access, derived state, and DTO mapping
to root modules. The runtime direction still starts at a service for most of these paths, but
ownership is structurally ambiguous and easy for routers to call directly, as BL-012 demonstrates.

Required boundary: move repository-backed helpers into `vat/services/` or
`vat/repositories/` according to responsibility. Keep pure date/period/serialization helpers
repository-free.

### BL-017 — timeline and notification query services are partly outside services

Evidence:

- `backend/app/timeline/services/timeline_service.py:25` and `:37` import root-level timeline
  aggregators.
- `backend/app/timeline/timeline_audit_aggregator.py:31-55` queries audit and user repositories.
- `backend/app/timeline/timeline_client_aggregator.py:17-23` queries timeline and legal-entity
  repositories.
- `backend/app/notifications/services/notification_send_service.py:27` and
  `notification_auto_send_service.py:20` import `NotificationContextResolver`.
- `backend/app/notifications/notification_context_resolver.py:54-63` constructs seven
  cross-domain repositories and then performs authorization-sensitive entity resolution.

Impact: these root modules are query/orchestration services by behavior but are not owned by the
service layer. In notifications, cross-domain reads and ownership validation are hidden behind a
generic “resolver” name.

Required boundary: move the aggregators/resolver under their domains' `services/` directories,
using `_service.py` names. If any logic is pure mapping, split it from the repository-backed
portion.

### BL-018 — domain guards and lookup helpers access repositories outside services

Evidence:

- `backend/app/annual_reports/annual_report_financial_line_helpers.py:54-64` queries a client and
  enforces mutation authorization.
- `backend/app/businesses/business_guards.py:9-35` queries a business and enforces create
  permission; it is imported by services across businesses, communications, documents, charges,
  notes, and signature requests.
- `backend/app/signature_requests/signature_request_validations.py:20-44` accepts repositories,
  performs lookups, and raises domain errors.
- `backend/app/signature_requests/signature_request_audit.py:63-142` orchestrates audit services,
  mutates persisted audit rows, and calls `db.flush()` from a root helper imported by four
  signature-request services.
- `backend/app/tax_calendar/tax_calendar_entry_lookup.py:20-32` constructs a repository directly.
- `backend/app/tax_calendar/tax_calendar_link_diagnostics.py:9` constructs a diagnostics
  repository directly.

Impact: fine-grained authorization and lookup/not-found decisions belong to services, but are
implemented in generic root helpers. Some pure guard functions are valid shared logic; the
violation is combining them with persistence access outside the service layer.

Required boundary: separate pure object guards from persistence-backed service methods. Move
repository-backed lookups and authorization checks into appropriately named services.
Seed-only diagnostics may instead become explicit tooling if they are not application behavior.

### BL-019 — reports export service raises FastAPI transport exceptions

Evidence:

- `backend/app/reports/services/report_reports_export_service.py:4` imports FastAPI
  `HTTPException`.
- `backend/app/reports/services/report_reports_export_service.py:42-51` and `:74-83` raise
  `HTTPException` from a service.

Impact: the service owns transport-layer error types, contrary to the explicit rule that services
raise `AppError` subclasses rather than FastAPI `HTTPException`. Non-HTTP callers cannot reuse the
export behavior without depending on FastAPI semantics.

Required boundary: introduce registered report export error codes and raise `AppError`
subclasses. The API error handler should remain responsible for mapping them to HTTP responses.

### BL-020 — client repository raises a domain not-found error

Evidence:

- `backend/app/clients/repositories/client_record_repository.py:11-12` imports `ErrorCode` and
  `NotFoundError`.
- `backend/app/clients/repositories/client_record_repository.py:113-125` combines a scalar lookup
  with the `CLIENT_RECORD_NOT_FOUND` domain decision.

Impact: repository callers cannot distinguish persistence absence from domain semantics, and the
repository owns an error decision that belongs to a service. This is the same responsibility
inversion already present more broadly in BL-009.

Required boundary: return `int | None` from the repository. The calling service should decide
whether absence is not-found, optional, forbidden, or otherwise invalid for its operation.

### BL-021 — advance-payment repository orchestrates a VAT repository

Evidence:

- `backend/app/advance_payments/repositories/advance_payment_turnover_lookup_repository.py:29-33`
  imports the VAT domain's repository and SQL helpers.
- `backend/app/advance_payments/repositories/advance_payment_turnover_lookup_repository.py:57-63`
  constructs `VatTurnoverRepository`.
- `backend/app/advance_payments/repositories/advance_payment_turnover_lookup_repository.py:87-93`
  calls the VAT repository and maps its result into advance-payment derived state.
- The module's `_to_resolution()` function at `:96-109` makes the business decision that fully
  filed VAT becomes `VAT_FILED` and partial coverage becomes `VAT_PENDING`.

Impact: a repository performs cross-domain orchestration and derived-state mapping. The backend
rule permits cross-domain repository joins only for scoping or typed projections; it assigns
cross-domain orchestration and derived state to services.

Required boundary: keep VAT's query contract in a VAT repository, but compose it from an
advance-payment service. Move `TurnoverResolution` and source classification to the service/domain
projection layer. SQL expressions required for filtered pagination may remain specialized read
repository primitives without constructing another repository.

### BL-022 — search repository imports a transport-schema enum

Evidence:

- `backend/app/search/repositories/search_match_repository.py:43` imports `SearchMatchType` from
  `app.search.schemas.search`.
- `backend/app/search/repositories/search_match_repository.py:51-63` exposes that schema-owned enum
  from its typed projection.

Impact: persistence depends on the API schema layer. The repository projection cannot be used
without importing transport contracts, reversing the intended service mapping direction.

Required boundary: move `SearchMatchType` to a neutral domain enum module and let both repository
projections and schemas depend on it, or map a repository-owned neutral value in `SearchService`.

### BL-023 — routers orchestrate multiple services for one endpoint operation

Evidence:

- `backend/app/clients/api/client_routes.py:185-192` updates through `ClientUpdateService` and then
  queries through `ClientQueryService`.
- `backend/app/clients/api/client_routes.py:220-223` restores through `ClientLifecycleService` and
  then queries through `ClientQueryService`.
- `backend/app/clients/api/client_routes_excel.py:40-42` composes `ClientQueryService` with
  `ClientExcelService`.
- `backend/app/clients/api/client_routes_excel.py:95-107` injects `CreateClientService` into
  `ClientExcelService` from the router.
- `backend/app/businesses/api/business_routes_client_businesses.py:40-48` creates through
  `BusinessService` and maps through `ClientBusinessService`.
- `backend/app/businesses/api/business_routes_client_businesses.py:93-101` repeats the split for
  update.
- `backend/app/annual_reports/api/annual_report_routes_detail.py:28-33` combines
  `AnnualReportService` and `AnnualReportDetailService`.
- `backend/app/annual_reports/api/annual_report_routes_detail.py:47-52` repeats that composition
  for update.
- `backend/app/advance_payments/api/advance_payment_routes_overview.py:107-143` calls two service
  methods and combines list rows with KPI results into one response.

Impact: routers coordinate multi-step application workflows and determine how independent service
results are combined. This violates the explicit “call one service method” route shape and makes
transaction, consistency, and response semantics transport-owned.

Required boundary: provide one endpoint-oriented orchestration method in the owning service.
Mutation services may return the complete response result or delegate internally to read/mapping
services. Export/import orchestration must likewise be owned below the router.

### BL-024 — routers own domain not-found, validation, and transition branching

Evidence:

- `backend/app/binders/api/binder_routes_list_get.py:71-76` decides that an absent service result
  is `BINDER_NOT_FOUND`.
- `backend/app/binders/api/binder_routes_list_get.py:88-93` repeats the not-found decision after
  deletion.
- `backend/app/annual_reports/api/annual_report_routes_create_read.py:114-117` maps an absent
  detail result to `ANNUAL_REPORT_NOT_FOUND`.
- `backend/app/annual_reports/api/annual_report_routes_create_read.py:129-132` repeats the
  decision after deletion.
- `backend/app/annual_reports/api/annual_report_routes_status.py:75-89` performs a transition,
  re-reads through a second service call, and owns the impossible/absent-result decision.
- `backend/app/reminders/api/reminder_routes_get.py:24-28` decides not-found in the router and uses
  a raw FastAPI `HTTPException`.
- `backend/app/users/api/user_routes_audit.py:37-41` validates the semantic ordering of a date
  range in the router.
- `backend/app/annual_reports/api/annual_report_routes_detail.py:29-33` decides how absence maps to
  an empty domain detail response.

Impact: domain semantics vary by caller because services return optional/bool values and routers
choose the error or fallback. This is business branching and invariant enforcement in the API
layer.

Required boundary: services should expose operation-specific methods that return the required
result or raise the registered `AppError`. Routers should retain request parsing and transport
mapping only. Optional lookup semantics may remain optional only where `None` is the documented
successful API result.

## Negative findings

The audit did not find service-layer SQL query execution through `Session.execute()`,
`Session.scalar()`, `Session.scalars()`, or legacy `Session.query()` in the scanned service
directories.

Service calls to `flush()` were found, but they are not violations: services are allowed to own
transaction boundaries and flush behavior. The finding in `client_graph_writer` is different
because that flush and its orchestration live in a repository module.

The import path
`vat/api/vat_serializers.py -> vat/vat_report_queries.py -> vat/repositories/*` was inspected.
The serializer currently calls only the pure deadline helper from that module, not its
repository-backed functions. It is therefore module coupling and poor separation, but not recorded
as an active runtime repository bypass in this audit.

## Recommended remediation order

1. Fix BL-009, the reverse repository-to-service dependency in the client graph writer.
2. Fix direct API repository access: BL-001 through BL-005.
3. Fix the grouped VAT bypass and background job boundary: BL-012 and BL-013.
4. Separate idempotency transport and persistence carefully: BL-014.
5. Move repository-backed response enrichment behind services: BL-006 and BL-007.
6. Normalize misplaced service behavior: BL-008 and BL-015 through BL-018.
7. Remove schema-to-repository coupling: BL-010.
8. Replace service-layer FastAPI exceptions: BL-019.
9. Remove repository-owned domain errors and schema dependencies: BL-020 and BL-022.
10. Move cross-domain turnover orchestration into a service: BL-021.
11. Collapse router-owned orchestration and domain branching: BL-023 and BL-024.
12. Decide and document the authentication boundary for BL-011 before changing it.

Each remediation should be a separate coherent change set with relevant backend tests. Moving
logic must preserve authorization, pagination, transaction ownership, response contracts, and
error codes.
