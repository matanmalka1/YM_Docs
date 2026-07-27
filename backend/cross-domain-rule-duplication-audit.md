## Scope

This file owns only:

- A code-verified audit of backend rules that are implemented independently in more than one
  domain, or that bypass an existing canonical cross-domain implementation.
- Evidence, behavioral differences, risks, and recommended ownership boundaries for each finding.

This file must not contain:

- New product behavior or domain invariants.
- A replacement for canonical architecture or domain documentation.
- An implementation plan presented as completed work.

Source of truth: reference

# Cross-Domain Rule Duplication Audit

Last verified against `backend/app`: 2026-07-27.

## Purpose

The backend is divided into vertical domain slices. A domain is expected to own behavior that is
specific to that domain, but it should not independently redefine a rule whose meaning is shared
across domains.

This audit identifies the latter cases. Its goal is not to remove every similar-looking function.
It records only rules for which at least one of the following is true:

1. two or more domains implement the same decision with materially identical code;
2. a canonical shared implementation exists, but one or more domains reimplement or bypass it;
3. the same external value is normalized independently at multiple boundaries that must agree;
4. a cross-domain policy has diverged in error type, error code, accepted state, or clock semantics.

The binding architecture rules remain in `docs/backend/architecture.md`,
`docs/architecture/api-contracts.md`, `docs/architecture/security.md`, and the relevant canonical
domain docs. This file is evidence and remediation guidance, not a new behavior owner.

## Method

The audit covered every Python file under `backend/app`, including routed domains, read-model
domains, infrastructure helpers, and shared modules.

The review combined:

- exact AST-body comparison for functions implemented in different top-level domains;
- searches for repeated guard, normalization, actor, clock, ownership, PATCH, and not-found
  patterns;
- comparison against existing shared helpers and mandatory architecture rules;
- manual inspection of each candidate to distinguish shared policy from legitimate domain logic.

Line numbers in this document identify the verified code as of the date above. They may shift after
unrelated edits; the path and symbol name are the durable identifiers.

## Classification

| Class | Meaning |
| --- | --- |
| Exact duplicate | Function bodies are identical apart from domain types or surrounding context. |
| Semantic duplicate | The same decision is implemented separately with small structural differences. |
| Canonical bypass | A shared owner exists, but a caller applies a local substitute. |
| Contract gap | Most domains use a shared contract while an outlier silently follows different rules. |
| Clock drift | Code obtains “now” locally instead of using the project-owned time boundary. |

## Summary

| ID | Rule family | Classification | Affected areas | Severity |
| --- | --- | --- | --- | --- |
| R1 | Due-date snapshot invariants | Exact duplicate | VAT, advance payments | High |
| R2 | Audit actor attribution | Exact + semantic duplicate | At least 12 domain/infrastructure paths | High |
| R3 | Client eligibility for mutation | Canonical bypass | Annual reports, VAT, notifications | Critical |
| R4 | Business/client ownership validation | Semantic duplicate + bypass | Businesses, communications, notes, VAT, charges, documents, signatures | Critical |
| R5 | Entity lookup and not-found mapping | Semantic duplicate | At least 8 domains | Medium |
| R6 | Email normalization | Exact semantic duplicate | Auth schemas, auth services, rate limiting | High |
| R7 | PATCH update contract | Contract gap | Charges | High |
| R8 | Current-time acquisition | Clock drift | Notifications, communications, signatures, VAT, annual reports, reports/exporters | High |
| R9 | Explicit-null rejection | Exact semantic duplicate | Nine update-schema families | Medium |
| R10 | Audit value serialization | Canonical bypass | Annual reports, charges, audit callers | High |
| R11 | Tax-rules financial access | Competing adapters | Common, VAT, clients, advance payments | Critical |
| R12 | Period/cadence validation | Semantic duplicate | Advance payments, tax calendar, VAT | High |
| R13 | Soft-delete lifecycle | Semantic duplicate + behavioral drift | Clients, businesses, reports, binders, charges, tasks, notes, communications | High |
| R14 | Client display identity resolution | Canonical bypass | Binders, annual reports, charges, reports/VAT export | Medium |
| R15 | Redundant base-repository overrides | Exact delegation + local reimplementation | Annual reports, advance payments, charges, businesses, communications, binders, notes, clients | Medium |
| R16 | Generic source-link integrity | Contract drift | Tasks, reminders, work queue | Critical |
| R17 | Required non-blank text | Canonical bypass + contract drift | Businesses, tasks, VAT, advance payments | High |
| R18 | Bulk identifier-list validation | Fragmented contract | Tasks, advance payments, charges, binders | High |
| R19 | Enum-string parsing | Semantic duplicate | Annual reports, reminders, signatures, timeline | Medium |
| R20 | Available-action capability rules | Mirrored business policy | Charges, VAT, tasks, work queue | Critical |
| R21 | Final/terminal status classification | Semantic duplicate + context drift | Annual reports, VAT, charges, binders, advance payments, work queue | High |
| R22 | User-search normalization | Fragmented contract | Work queue, timeline, global search, SQL list searches | Medium |

## R1 — Due-date snapshot invariants

### Shared rule being recreated

Entities that preserve both an original and an effective due date enforce the following rule set:

1. initialize the effective snapshot from the original date;
2. do not allow a non-null original date to change after it has been set;
3. require a non-blank reason when the effective date differs from the original date;
4. enforce those rules on both insert and update through SQLAlchemy events.

### Implementations

| Domain | Evidence |
| --- | --- |
| Advance payments | `backend/app/advance_payments/models/advance_payment_due_date_snapshot_events.py:8-48` |
| VAT | `backend/app/vat/models/vat_due_date_snapshot_events.py:8-46` |

The following functions have the same bodies in both modules:

- `_require_override_reason`;
- `_ensure_original_immutable`;
- `_before_insert`;
- `_before_update`.

Both modules also register an `active_history=True` listener for `due_date_original`.

### Existing difference

`_default_due_date_snapshots` is not fully identical:

- advance payments first copies legacy `due_date` into `due_date_original` when the original is
  absent, then initializes `due_date_effective`;
- VAT only initializes `due_date_effective` when `due_date_original` exists.

That legacy-source difference is entity-specific. The immutability, override-reason, event, and
effective-date rules are not.

### Risk

- A future correction to immutability or blank-reason handling can be applied to one tax domain but
  not the other.
- A third obligation type can copy the event module again and create another policy owner.
- The invariant currently raises raw `ValueError`; duplicating it makes later conversion to a
  structured domain error harder to perform consistently.

### Recommended ownership boundary

Own the invariant in one shared due-date snapshot helper that accepts the mapped model and, if
needed, an entity-specific original-date initializer. Keep only the legacy `due_date` fallback in
the advance-payment adapter.

Do not place tax-specific deadlines or deadline calculation rules in that helper. It should own
only snapshot mechanics and mutation invariants.

## R2 — Audit actor attribution

### Shared rule being recreated

Multiple write services infer audit identity using this rule:

```text
actor_id is None
  -> actor_type = "system"
  -> actor_display_name = supplied name, otherwise "מערכת"

actor_id is not None
  -> preserve the audit writer's default user actor type
  -> pass actor_display_name only
```

This is a cross-domain audit policy: the same absence of a user ID must mean the same actor type in
every audit row.

### Exact `_actor_kwargs` copies

The same function body exists in these seven modules:

| Domain | Evidence |
| --- | --- |
| Communications | `backend/app/communications/services/correspondence_service.py:42-48` |
| Reminders | `backend/app/reminders/services/reminder_service.py:32-38` |
| Advance payments | `backend/app/advance_payments/services/advance_payment_service.py:147-153` |
| Notes | `backend/app/notes/services/note_entity_note_service.py:30-36` |
| Authority contacts | `backend/app/authority_contacts/services/authority_contact_service.py:29-35` |
| Invoices | `backend/app/invoices/services/invoice_service.py:26-32` |
| Tasks | `backend/app/tasks/services/task_service.py:86-92` |

Each also declares its own `_SYSTEM_ACTOR_DISPLAY = "מערכת"` constant.

### Additional equivalent implementations

| Area | Form | Evidence |
| --- | --- | --- |
| Permanent documents | Equivalent `_actor_kwargs`, with `actor_display_name` parameter naming | `backend/app/documents/permanent_documents/services/permanent_document_service.py:63,77-82` |
| Notifications | Inline conditional dictionary | `backend/app/notifications/services/notification_send_service.py:461-465` |
| Reminder executor | Direct system actor values in multiple writes | `backend/app/reminders/services/reminder_executor_service.py:102-103,230-231` |
| Tax calendar | Direct `actor_type="system"` | `backend/app/tax_calendar/services/tax_calendar_entry_service.py:301` |
| Signature requests | Direct system actor values | `backend/app/signature_requests/services/signature_request_service.py:136`; `backend/app/signature_requests/signature_request_audit.py:135` |

### Behavioral ambiguity

Several other services pass `actor_id=None` and/or `actor_display_name` without explicitly setting
`actor_type`. Whether those rows become user or system actors then depends on audit-writer defaults
instead of one visible rule. Examples occur in annual reports, businesses, clients, charges, and
users.

The duplicate helper therefore masks a wider ownership problem: actor inference is decided partly
by callers and partly by the audit subsystem.

### Risk

- System actions can be recorded as user actions when a caller omits the local conditional.
- The Hebrew fallback name can change in only some domains.
- New actor kinds, such as an external signer or background job, require edits across unrelated
  services.
- Auditing and timeline consumers cannot safely treat `actor_type` as uniformly derived.

### Recommended ownership boundary

The audit domain should own actor construction or inference. Callers should pass a typed actor
context, or call one audit-owned helper. Domain services should not decide how `None` maps to
`actor_type`.

## R3 — Client eligibility for mutation

### Canonical rule

The canonical runtime guard is:

`backend/app/clients/guards/client_record_guards.py:33-50`

`assert_client_record_is_active` uses an allowlist:

- only `ClientStatus.ACTIVE` is eligible;
- every other current or future status is rejected;
- rejection is a `ConflictError`;
- the common error code is `ErrorCode.CLIENT_RECORD_CLOSED`;
- closed and frozen states differ only in the user-facing message.

Its module explicitly describes itself as “the single client-eligibility rule every domain blocks
on” and requires parity with the SQL predicate in
`backend/app/clients/repositories/client_active_scope.py`.

### Correct canonical consumers

| Domain | Evidence |
| --- | --- |
| Charges | `backend/app/charges/services/charge_billing_service.py:87` |
| Annual-report creation | `backend/app/annual_reports/services/annual_report_create_service.py:61` |
| Binder intake | `backend/app/binders/services/binder_intake_service.py:111` |
| VAT client context | `backend/app/vat/services/vat_client_context_service.py:23` |
| Advance payments | `backend/app/advance_payments/services/advance_payment_service.py:164-168` |

### Canonical bypasses

#### Annual-report financial mutations

`backend/app/annual_reports/annual_report_financial_line_helpers.py:54-64` reimplements the state
test:

- `CLOSED` raises `ForbiddenError` with `ErrorCode.CLIENT_CLOSED`;
- `FROZEN` raises `ForbiddenError` with `ErrorCode.CLIENT_FROZEN`;
- an unknown future status is implicitly allowed.

This differs from the canonical rule in status code semantics, error-code semantics, and
future-status behavior.

#### VAT invoice creation

`backend/app/vat/services/vat_data_entry_invoices_service.py:79-90` rejects only
`ClientStatus.CLOSED`.

Consequences:

- a frozen client is not rejected at this site;
- a future non-active status is implicitly allowed;
- the local error is `AppError` with `ErrorCode.VAT_CLIENT_CLOSED`, not the canonical conflict.

#### Notification policy

`backend/app/notifications/services/notification_policy_service.py:60-77` treats `FROZEN` and
`CLOSED` as a special pair and then allows trigger-specific exceptions through
`_FROZEN_CLOSED_ALLOWED`.

This may be legitimate notification-specific product policy. It must nevertheless be explicit as
an exception owned by the notifications domain rather than presented as a substitute for generic
mutation eligibility. It should not be mechanically replaced without confirming the domain rule.

### Risk

- The same client can be eligible in one mutation flow and ineligible in another.
- HTTP status and error codes vary for the same underlying resource state.
- Blocklists fail open when a new enum value is added.
- SQL list eligibility and service mutation eligibility can disagree.

### Recommended ownership boundary

All ordinary client-scoped mutations should call `assert_client_record_is_active`. Any operation
that intentionally works for a closed or frozen client should have a narrowly named, domain-owned
capability rule documenting the exception.

## R4 — Business/client ownership validation

### Shared rule being recreated

For a client-scoped operation with a `business_id`:

1. the client record must exist;
2. the business must exist;
3. `business.legal_entity_id` must equal the client's `legal_entity_id`;
4. failure must not expose or mutate a business belonging to another client.

The low-level comparison already exists in:

`backend/app/businesses/business_guards.py:38-45`

as `assert_business_belongs_to_legal_entity`.

### Implementations using the shared comparison but rebuilding orchestration

| Domain | Local work repeated | Evidence |
| --- | --- | --- |
| Businesses | Load client, then call shared assertion | `backend/app/businesses/services/business_client_business_service.py:84-87` |
| Communications | Load business and map not-found locally, then call shared assertion | `backend/app/communications/services/correspondence_service.py:68-73` |
| Notes | Load business and client, map both errors, then call shared assertion | `backend/app/notes/services/note_business_note_service.py:99-108` |
| Charges | Validate optional business scope against supplied legal entity | `backend/app/charges/services/charge_billing_service.py:60-70` |
| Permanent documents | Validate document business context against the client | `backend/app/documents/permanent_documents/services/permanent_document_service.py:160-169` |
| Signature requests | Validate request business against client legal entity | `backend/app/signature_requests/services/signature_request_creation_service.py:64-72` |
| Business service | Repeats the same relationship check in mutation flow | `backend/app/businesses/services/business_service.py:149-155` |

### Direct comparison bypasses

VAT reimplements the relationship without the shared assertion:

- invoice creation:
  `backend/app/vat/services/vat_data_entry_invoices_service.py:92-98`;
- invoice update:
  `backend/app/vat/services/vat_data_entry_invoice_update_service.py:87-98`.

The two VAT paths also disagree with each other:

| Path | Failure code/status |
| --- | --- |
| Create | `ErrorCode.BUSINESS_ACTIVITY_WRONG_CLIENT` through `AppError` |
| Update | `ErrorCode.BUSINESS_ACTIVITY_NOT_FOUND`, explicit HTTP 404 |

### Other client-child ownership copies

The same broader rule—load a child and verify `child.client_record_id`—is recreated for several
entity kinds:

- correspondence entry:
  `backend/app/communications/services/correspondence_service.py:95-103`;
- authority contact:
  `backend/app/communications/services/correspondence_service.py:83-93`;
- binder handover:
  `backend/app/binders/services/binder_handover_service.py:76`;
- notification policy for reports, VAT items, charges, and signatures:
  `backend/app/notifications/services/notification_policy_service.py:170,200,219,253,281,293`;
- notification context resolution:
  `backend/app/notifications/notification_context_resolver.py:148-188`;
- VAT amendment relationship:
  `backend/app/vat/services/vat_filing_service.py:41`.

These are not necessarily one generic helper candidate: entity-specific existence and disclosure
semantics may differ. They are included because the ownership rule is repeatedly reconstructed and
must be reviewed as one security boundary.

### Risk

- IDOR protection depends on every CRUD path remembering the comparison.
- Create and update can return different errors for the same invalid relationship.
- A direct comparison can omit active-client scoping or soft-delete behavior embedded in a
  canonical repository/guard.
- Combining “not found” and “belongs to another client” inconsistently may leak entity existence.

### Recommended ownership boundary

Create a business-domain or client-domain ownership resolver that loads both resources and returns
the validated business. It should expose deliberately named variants if disclosure policy differs,
for example a path-scoped 404 variant versus an explicit validation-error variant.

Do not create one untyped universal `belongs_to` helper for every entity. Business-to-legal-entity
ownership has a stable shared shape; arbitrary child ownership does not.

## R5 — Entity lookup and not-found mapping

### Existing shared implementation

`BaseService` owns:

- `_get_or_raise(repo, entity_id, error_code)`;
- `_get_or_raise_for_update(repo, entity_id, error_code)`.

Evidence: `backend/app/common/services/base_service.py:52-70`.

### Local implementations

| Domain | Symbol | Evidence |
| --- | --- | --- |
| Advance payments | `_get_record_or_raise` | `backend/app/advance_payments/services/advance_payment_service.py:155-162` |
| Annual reports | `_get_or_raise` | `backend/app/annual_reports/services/annual_report_base_service.py:33-40` |
| Annual reports | `_get_or_raise_for_update` | `backend/app/annual_reports/services/annual_report_status_service.py:64-72` |
| Annual reports | `_get_report_or_raise` | `backend/app/annual_reports/services/annual_report_charge_service.py:25`; `annual_report_financial_line_service.py:60`; `annual_report_tax_service.py:71` |
| Businesses | `get_business_or_raise` | `backend/app/businesses/business_guards.py:9`; `backend/app/businesses/services/business_service.py:121` |
| Clients | `get_client_or_raise` | `backend/app/clients/services/client_service.py:10` |
| Communications | `_get_client_record_or_raise`, `_get_entry_or_raise` | `backend/app/communications/services/correspondence_service.py:75,95` |
| Notes | `_get_or_raise` | `backend/app/notes/services/note_entity_note_service.py:70` |
| Signature requests | `get_or_raise`, token variants | `backend/app/signature_requests/signature_request_validations.py:20-45` |
| Users | `get_user_or_raise` | `backend/app/users/services/user_management_service.py:29-33` |

### Important distinctions

Not every local function can be replaced mechanically:

- token lookup is not ID lookup;
- a client-scoped lookup may intentionally collapse wrong-owner and missing-resource cases;
- domain-specific Hebrew messages are part of the API behavior;
- row-locking must remain explicit for lifecycle mutations.

The duplicated rule is the orchestration skeleton—repository lookup, null test, structured
not-found—not the domain-specific message.

### Risk

- Some services inherit `BaseService`, while others do not, creating two conventions.
- Locking can be forgotten when a local helper is copied from a non-locking path.
- Error-code formatting and Hebrew messages drift.
- Soft-deleted identity lookup can accidentally switch between `get_by_id` and
  `include_deleted=True` behavior.

### Recommended ownership boundary

Choose one service-level lookup convention. Either make relevant services inherit `BaseService`,
or use a shared standalone lookup primitive that accepts a message factory. Preserve explicitly
named variants for row locks, tokens, owner-scoped lookup, and include-deleted lookup.

## R6 — Email normalization

### Shared rule being recreated

Every email identity boundary currently normalizes strings with:

```python
email.strip().lower()
```

### Implementations

| Boundary | Evidence |
| --- | --- |
| Rate-limit key | `backend/app/middleware/rate_limiting.py:33-34` |
| Login request schema | `backend/app/users/schemas/user_auth.py:13-18` |
| Forgot-password request schema | `backend/app/users/schemas/user_auth.py:35-40` |
| Authentication lookup | `backend/app/users/services/user_auth_service.py:59-62` |
| Password-reset lookup | `backend/app/users/services/user_password_reset_service.py:45-53` |

The schema and service layers therefore normalize the same login/reset inputs twice. Rate limiting
has a separate implementation that must remain identical to avoid assigning multiple rate-limit
keys to one account.

### Risk

- Changing Unicode or case-normalization policy in only one path can make the rate-limit identity
  differ from the authentication identity.
- A future user-creation or email-update path can store a value using another normalization rule.
- Repeated normalization hides which layer owns the persisted canonical form.

### Recommended ownership boundary

The users/auth boundary should own one pure `normalize_email` function. Request schemas,
authentication, password reset, user creation/update, repository lookup, and rate-limit key
generation should all call it.

The canonical function should be chosen before changing behavior: consolidating the existing
`strip().lower()` rule is mechanical; switching to `casefold()` or another email policy is a
separate contract decision.

## R7 — PATCH update contract

### Canonical shared contract

`NonEmptyUpdateMixin` in `backend/app/core/schemas/validation.py:4-23` enforces:

1. unknown fields are rejected with `extra="forbid"`;
2. an empty `{}` PATCH body is rejected;
3. field presence is based on `model_fields_set`, preserving the distinction between omitted and
   explicit `null`.

This matches the binding API conventions referenced by
`docs/architecture/update-request-conventions.md` and `docs/architecture/api-contracts.md`.

### Conforming domain schemas

The mixin is used by update requests in:

- communications;
- users;
- authority contacts;
- permanent documents;
- annual-report details, deadlines, financial lines, and annex data;
- tasks;
- notes;
- binders;
- businesses;
- VAT work items and invoices;
- clients;
- advance payments.

### Outlier

`ChargeUpdateRequest` inherits directly from `BaseModel`:

`backend/app/charges/schemas/charge.py:21-45`.

It locally implements only explicit-null rejection for selected non-nullable fields. The route
passes `model_dump(exclude_unset=True)`:

`backend/app/charges/api/charge_routes.py:68-79`.

Consequences:

- `{}` is accepted by schema validation instead of receiving the common 422;
- unknown fields follow Pydantic's default ignore behavior instead of being rejected;
- the domain has recreated only one fragment of the common PATCH rule.

### Risk

- Removed or misspelled fields can be silently swallowed by the charge endpoint.
- Clients receive different empty-PATCH behavior for charges than for other resources.
- Tests may mistake a successful no-op for a valid mutation.

### Recommended ownership boundary

`ChargeUpdateRequest` should use `NonEmptyUpdateMixin` while retaining its field-specific
explicit-null validators. This is a contract alignment, not a new charge-domain rule.

## R8 — Current-time acquisition

### Canonical rule

`docs/backend/architecture.md` requires:

- `app.utils.time_utils.israel_today()` for business-local dates;
- `utcnow()` for naive UTC database columns;
- `utcnow_aware()` for timezone-aware UTC timestamps.

The implementations live in `backend/app/utils/time_utils.py:13-32`.

### Business-policy clock bypasses

These sites use `datetime.now` directly for decisions or validation:

| Area | Use | Evidence | Expected boundary |
| --- | --- | --- | --- |
| Notifications | “days since last notification” and signature expiry | `backend/app/notifications/services/notification_policy_service.py:183,265,297` | `utcnow_aware()` |
| Communications | Reject future `occurred_at` | `backend/app/communications/schemas/correspondence.py:12-17` | `utcnow_aware()` or injected validation clock |
| Signature requests | Expiry-derived schema state | `backend/app/signature_requests/schemas/signature_request.py:127` | `utcnow_aware()` |
| VAT | Derive “today” for report queries | `backend/app/vat/vat_report_queries.py:26` | Determine whether rule is Israel-local; if so `israel_today()` |
| Annual reports | Staleness and retention cutoffs | `backend/app/annual_reports/repositories/annual_report_report_lifecycle_repository.py:75,96` | Service-computed cutoff using `utcnow_aware()` |

The annual-report repository case also crosses a layer boundary: business cutoffs are calculated
inside a repository, while repositories should own database access rather than business decisions.

### Presentation/export clock bypasses

These uses do not necessarily change business eligibility, but they independently choose a local
timezone and timestamp format:

- reports PDF exporter:
  `backend/app/reports/exporters/pdf_exporter.py:38,106,133,207`;
- reports Excel exporter:
  `backend/app/reports/exporters/excel_exporter.py:88,166`;
- annual-report PDF builder:
  `backend/app/annual_reports/annual_report_pdf_builder.py:189`;
- VAT PDF exporter:
  `backend/app/vat/exporters/pdf_exporter.py:65,219,229`;
- VAT Excel exporter:
  `backend/app/vat/exporters/excel_exporter.py:84`;
- shared Excel utility:
  `backend/app/utils/excel.py:82,96`.

For user-visible Israeli reports, naive `datetime.now()` currently means “whatever timezone the
process uses,” not necessarily Israel time. That can produce the wrong displayed date or filename
around midnight or when the runtime is configured for UTC.

### Exclusions

The following were reviewed but are not findings in this audit:

- `datetime.now(...)` inside seed builders, because seed generation is non-domain tooling;
- the direct `datetime.now(...)` calls inside `time_utils.py`, because that module is the canonical
  clock implementation itself.

### Risk

- Date-based behavior differs between local development and a UTC production runtime.
- Tests must patch many modules instead of one clock boundary.
- A business-local date can roll over two or three hours earlier than intended.
- Export timestamps can disagree with audit timestamps for the same operation.

### Recommended ownership boundary

Route business clocks through `time_utils`. Add an explicit Israel-local datetime helper if export
presentation genuinely needs local wall-clock time; do not use naive process-local
`datetime.now()` as that helper.

## R9 — Explicit-null rejection in update schemas

### Shared rule being recreated

`NonEmptyUpdateMixin` owns the common PATCH envelope, but each update schema separately implements
the following field-level rule:

```text
for every field backed by a non-nullable column or required relationship:
  if the field was explicitly sent and its value is null:
    reject the request
```

The implementations use `model_fields_set` correctly, but the loop and Hebrew error are repeated.

### Implementations

| Domain/schema | Non-nullable fields enumerated locally | Evidence |
| --- | --- | --- |
| Communications | `correspondence_type`, `subject`, `occurred_at` | `backend/app/communications/schemas/correspondence.py:49-54` |
| Users | `role`, `email`, `full_name` | `backend/app/users/schemas/user_management.py:29-36` |
| Tasks | `title`, `priority` | `backend/app/tasks/schemas/task.py:49-54` |
| Businesses | `business_name`, `status` | `backend/app/businesses/schemas/business_schemas.py:63-68` |
| Authority contacts | `contact_type`, `name` | `backend/app/authority_contacts/schemas/authority_contact.py:27-32` |
| Binder intake | dates, actors, transfer targets, association lists | `backend/app/binders/schemas/binder.py:145-160` |
| Advance payments | `paid_amount`, `expected_amount` | `backend/app/advance_payments/schemas/advance_payment.py:180-186` |
| Annual-report income | `source_type`, `amount` | `backend/app/annual_reports/schemas/annual_report_financials.py:29-34` |
| Annual-report expenses | `category`, `amount`, `recognition_rate` | `backend/app/annual_reports/schemas/annual_report_financials.py:74-81` |

VAT invoice updates and permanent-document updates enforce related explicit-null rules with
field-specific validators rather than the same loop:

- `backend/app/vat/schemas/vat_invoice_update.py:52-63`;
- `backend/app/documents/permanent_documents/schemas/permanent_document.py:46-51`.

Charges also implement a field-level explicit-null validator, but—as documented in R7—do not
inherit the rest of the common update contract:

`backend/app/charges/schemas/charge.py:35-45`.

### What is shared and what remains domain-owned

The set of nullable fields belongs to each request contract. A shared helper must not infer
nullability from Python's `T | None`, because partial-update schemas use `None` as the default even
for non-nullable database fields.

The repeated mechanism is:

- detect explicit presence through `model_fields_set`;
- reject `None`;
- render one consistent validation error.

### Risk

- A newly added non-nullable update field can be omitted from the local tuple.
- Error text can drift across domains.
- A developer may incorrectly test `value is None` without checking field presence, thereby
  rejecting omitted fields.
- Database nullability and request nullability can silently diverge.

### Recommended ownership boundary

Add one schema-layer helper or mixin utility that accepts an explicit tuple of non-nullable field
names. Each domain continues to declare its field tuple; the common layer owns presence detection
and error construction.

## R10 — Audit value serialization

### Canonical implementation

`EntityAuditWriter` already normalizes audit payloads recursively:

- `Enum` to `.value`;
- `date` and `datetime` to ISO strings;
- `Decimal` to strings;
- nested dictionaries, lists, and tuples recursively.

Evidence:

`backend/app/audit/services/audit_entity_audit_writer_service.py:239-263`.

All `old_value`, `new_value`, and metadata passed through the writer therefore have one JSON-safe
serialization boundary.

### Canonical bypasses

#### Annual reports

`backend/app/annual_reports/services/annual_report_detail_service.py:66-71` defines `_audit_value`
that converts enums and dates before passing values to the audit writer.

#### Charges

`backend/app/charges/services/charge_billing_service.py:37-43` defines another `_audit_value` that
converts enums and decimals.

The two local serializers are incomplete in different ways:

| Serializer | Enum | Date/datetime | Decimal | Recursive containers |
| --- | --- | --- | --- | --- |
| Audit-owned canonical serializer | Yes | Yes | Yes | Yes |
| Annual-report `_audit_value` | Yes | Yes | No | No |
| Charge `_audit_value` | Yes | No | Yes | No |

### Additional local normalization

Multiple domain `_audit_snapshot` and `_audit_metadata` functions build values that rely on the
writer's recursive normalization. That is correct. The local `_audit_value` functions are
therefore redundant and make callers believe they must predict storage serialization themselves.

### Risk

- Audit diffs can encode the same type differently depending on which domain preprocessed it.
- A nested value is normalized only by the canonical writer, while a caller may compare it against
  a prematurely converted scalar.
- Changes to audit storage format require locating domain-local serializers.
- Domain code takes ownership of an audit persistence concern.

### Recommended ownership boundary

Callers should pass typed values to `EntityAuditWriter`. If callers need normalized values before
the append call—for example to calculate a diff—the audit domain should expose one public
normalization or diff-construction API rather than duplicating partial serializers.

## R11 — Tax-rules financial access

### Shared rule being recreated

The current VAT rate and other year-specific tax financials come from the standalone `tax_rules`
package. There should be one adapter defining:

- which package import path is canonical;
- whether a missing key is exceptional or optional;
- the returned type;
- conversion to `Decimal`;
- whether errors propagate.

### Competing adapters

#### Common adapter

`backend/app/common/integrations/tax_rules_financials.py:4-10`:

- imports `get_financial` from `tax_rules.registry`;
- reads `"vat_rate_percent"`;
- converts the value to `Decimal`;
- catches every `Exception`;
- returns `None` on any failure.

#### VAT adapter

`backend/app/vat/integrations/tax_rules_financials.py:4-23`:

- imports `get_financial` from the package root;
- exposes a generic `get_financial_value`;
- returns the package result without conversion;
- exposes its own `get_vat_rate_percent`;
- lets errors propagate;
- returns the raw `.value` type.

### Consumers

| Consumer | Adapter | Evidence |
| --- | --- | --- |
| Advance-payment constants | Common optional/`Decimal` adapter | `backend/app/advance_payments/advance_payment_constants.py:6-12` |
| VAT amount splitting | VAT raw adapter, then local `Decimal(str(...))` | `backend/app/vat/vat_amounts.py:5,19,35` |
| VAT exceptional-invoice threshold | VAT generic adapter | `backend/app/vat/vat_data_entry_common.py:9,97` |
| VAT exempt ceiling | VAT generic adapter | `backend/app/vat/vat_data_entry_common.py:117` |
| VAT invoice update threshold | VAT generic adapter | `backend/app/vat/services/vat_data_entry_invoice_update_service.py:16,162` |
| Client creation policy | Imports the VAT-domain adapter cross-domain | `backend/app/clients/client_create_policy.py:6,31` |

### Behavioral conflict

This is not merely two names for the same helper:

- one path silently converts all import, configuration, lookup, and type errors to `None`;
- another path raises;
- one guarantees `Decimal | None`;
- another exposes the tax-rules package's raw value;
- clients depend on a VAT-owned integration for a client-domain creation rule.

The broad `except Exception: return None` also conflicts with the project rule against hidden
fallback behavior. A broken tax-rules package is indistinguishable from “VAT rate is not
configured.”

### Risk

- The same missing tax rule can disable a feature silently in one domain and fail loudly in
  another.
- Financial calculations can use inconsistent numeric types.
- Package API changes can break one import path but not the other.
- Cross-domain imports make the VAT slice an accidental owner of client creation policy.

### Recommended ownership boundary

Create one common integration adapter for all tax-rules financial access. It should return explicit
typed values—preferably `Decimal` for monetary/rate values—and use a narrow, documented error
policy. Optional lookup, if genuinely required, should be a separately named method rather than a
broad exception fallback.

Any semantic change to tax-rule absence or fallback behavior must follow
`docs/project/tax-rules-config.md`.

## R12 — Period and cadence validation

### Shared rule being recreated

Periodic tax obligations use a `YYYY-MM` start period and a `period_months_count` of 1 or 2. For a
two-month cadence, valid start months are odd months: 1, 3, 5, 7, 9, and 11.

The common parser already owns basic period extraction:

- `parse_period_year`;
- `parse_period_month`;
- month-shifting and covered-period helpers.

Evidence: `backend/app/common/period_utils.py:21-63`.

The cadence validity rule itself is still implemented in several places.

### Implementations

#### Advance-payment request schema

`backend/app/advance_payments/schemas/advance_payment.py:144-155`:

- checks `SUPPORTED_PERIOD_MONTH_COUNTS`;
- parses the month through the common parser;
- tests membership in `BIMONTHLY_START_MONTHS`;
- raises a Pydantic `ValueError`.

#### Advance-payment service

`backend/app/advance_payments/services/advance_payment_service.py:182-190` repeats the same cadence
rule and raises `ConflictError` with `ADVANCE_PAYMENT_INVALID_PERIOD`.

#### Tax-calendar materialization

`backend/app/tax_calendar/services/tax_calendar_materialization_service.py:184-193`:

- parses the period locally with `split("-")`;
- checks the year/month shape locally;
- rejects an even month when `months == 2`;
- raises raw `ValueError`.

#### Tax-calendar entry model

`backend/app/tax_calendar/models/tax_calendar_entry.py:126-180` independently validates:

- period shape;
- `period_months_count in (1, 2)`;
- obligation-type consistency;
- monthly/bimonthly entry requirements.

#### Tax-calendar entry service

`backend/app/tax_calendar/services/tax_calendar_entry_service.py:174-181` uses
`(month - 1) % period_months_count` as another alignment rule.

#### VAT intake

`backend/app/vat/services/vat_intake_service.py:33-44` validates period alignment against VAT type
and rejects an even month for bimonthly VAT clients.

### Important distinction

Validation at schema, service, and persistence boundaries can be intentional defense in depth.
The duplication finding is not “validate only once.” The finding is that every boundary encodes
the odd-month and supported-cadence formula independently instead of calling one pure predicate.

### Risk

- Adding a supported cadence requires updating constants, schemas, services, and model listeners.
- Period parsing accepts or rejects different malformed strings depending on entry point.
- Callers receive Pydantic errors, conflicts, or raw `ValueError` for the same invalid pair.
- VAT and advance-payment cadence rules can drift even though both feed tax-calendar
  materialization.

### Recommended ownership boundary

Move parsing and cadence alignment into pure common period helpers, for example a function that
validates `(period, period_months_count)` and returns the parsed year/month. Boundary layers should
translate the shared validation result into their appropriate error type without recreating the
formula.

Keep obligation-specific decisions—such as whether a client is monthly or bimonthly—in the owning
domain.

## R13 — Soft-delete lifecycle

### Shared mechanics being recreated

Soft-deletable domain services repeatedly implement:

1. load the active or include-deleted entity;
2. decide whether missing/already-deleted is not-found or conflict;
3. call repository `soft_delete` or `restore`;
4. write a delete/restore audit event;
5. return an entity, boolean, or no content.

`BaseRepository` owns the persistence mechanics, and `EntityAuditWriter` owns standard
`record_delete`/`record_restore` events. The service contract around them is not consistent.

### Implementations and differences

| Domain | Missing delete | Delete audit | Restore support | Evidence |
| --- | --- | --- | --- | --- |
| Clients | Raises not-found; already deleted is also not-found | Yes | Yes; active-ID conflict check | `backend/app/clients/services/client_lifecycle_service.py:25-68` |
| Businesses | Raises not-found | Yes | Yes; advisor-only | `backend/app/businesses/services/business_lifecycle_service.py:29-64` |
| Annual reports | Returns `False` | Yes | No service restore | `backend/app/annual_reports/services/annual_report_service.py:60-80` |
| Binders | Returns `False` | No audit at this site | No service restore | `backend/app/binders/services/binder_service.py:39-43` |
| Charges | Raises not-found; status-gated | Yes | No service restore | `backend/app/charges/services/charge_billing_service.py:263-286` |
| Advance payments | Raises owner-scoped not-found; reason required | Custom audit action | No service restore | `backend/app/advance_payments/services/advance_payment_service.py:600-635` |
| Communications | Owner-scoped lookup then soft delete | Yes through local audit call | No restore | `backend/app/communications/services/correspondence_service.py:190-211` |
| Notes | Owner-scoped lookup then soft delete | Yes through local audit call | No restore | `backend/app/notes/services/note_entity_note_service.py:155-176` |
| Tasks | Directly assigns `deleted_at`/`deleted_by` instead of repository helper | Domain audit path differs | Restore-like `get` rejects deleted rows | `backend/app/tasks/services/task_service.py:438-460` |

### What is domain-specific

The following must remain domain-owned:

- which statuses permit deletion;
- whether a delete requires a reason;
- cascades such as canceling signature requests;
- uniqueness checks during restore;
- who is authorized to restore.

The duplicated cross-domain mechanics are resource-state interpretation, repository operation,
standard audit event construction, and return/error convention.

### Risk

- A deletion can be unaudited in one domain and audited in another.
- Routes can accidentally interpret `False` as success or produce different 404 behavior.
- Direct timestamp assignment can bypass repository-level soft-delete behavior.
- Already-deleted resources alternate between 404 and conflict without an explicit contract.
- Restore authorization is enforced inside one service but may be assumed to live at route level
  elsewhere.

### Recommended ownership boundary

Document one backend soft-delete lifecycle convention covering missing/already-deleted behavior,
standard audit requirements, and repository use. Provide shared mechanics only after domain hooks
for status checks, cascade work, reason metadata, and restore uniqueness have been identified.

Do not mechanically replace the domain services with a generic CRUD service.

## R14 — Client display identity resolution

### Canonical implementation

`ClientIdentityRepository.get_display_map` returns, for a batch of client-record IDs:

- official client name;
- office client number;
- legal-entity ID number;
- consistent deleted-client filtering controlled by `include_deleted`.

Evidence:

`backend/app/clients/repositories/client_identity_repository.py:19-62`.

It is already consumed by notifications, reminders, tasks, work queue, and advance-payment
reports.

### Canonical bypasses

#### Binders

`backend/app/binders/services/binder_list_service.py:32-54` independently:

- loads client records;
- extracts legal-entity IDs;
- loads legal entities;
- builds three maps for the same display fields.

#### Annual reports

`backend/app/annual_reports/services/annual_report_base_service.py:42-60` performs the same
two-stage load. Response mapping at lines 71-81 then copies the same three identity fields.

#### Charges

`backend/app/charges/services/charge_query_service.py:105-131` separately:

- loads client records;
- calls `get_full_records_bulk`;
- builds charge-keyed name and office-number maps.

This path uses `full_name`, while the canonical repository defines `client_name` as
`LegalEntity.official_name`. The difference may be historical or intentional, but it means
“client_name” does not have one stable source.

#### VAT and report export paths

Several export/report services directly load `ClientRecord` followed by `LegalEntity`, including:

- `backend/app/vat/services/vat_export_service.py:32-39`;
- `backend/app/reports/services/report_service.py:16-17`.

Single-record export lookup may not need batch mapping, but it still recreates the identity-source
decision.

### Risk

- `client_name` can mean official legal-entity name in one response and a derived full name in
  another.
- Deleted-client visibility differs depending on whether the canonical `include_deleted` option is
  used.
- Every list service rebuilds batching and missing-entity behavior.
- Client identity schema changes require edits across read domains.

### Recommended ownership boundary

Use `ClientIdentityRepository` for the canonical display triple and add a single-record method if
needed. If charges genuinely require a different personal/display name, name that field and
capability explicitly rather than returning it as the same `client_name` concept.

## R15 — Redundant base-repository overrides

### Canonical rule

`BaseRepository` already owns:

- `get_by_id`;
- keyword-field `create`;
- generic `update`;
- `soft_delete`;
- `hard_delete`;
- timestamp and `deleted_by` handling.

Evidence: `backend/app/common/repositories/base_repository.py:44-123`.

`docs/backend/architecture.md` explicitly prohibits overriding a `BaseRepository` method when the
inherited behavior is identical. An override is allowed only when it adds real, differing
behavior.

### Pure delegation overrides

The following overrides add no domain behavior and only call the base soft-delete implementation:

| Domain | Evidence |
| --- | --- |
| Annual reports | `backend/app/annual_reports/repositories/annual_report_report_repository.py:239-240` |
| Advance payments | `backend/app/advance_payments/repositories/advance_payment_repository.py:200-201` |
| Charges | `backend/app/charges/repositories/charge_repository.py:373-374` |

They call `_soft_delete_entity`, which is itself only a protected bridge to
`BaseRepository.soft_delete`:

`backend/app/common/repositories/base_repository.py:104-105`.

### Local reimplementations of generic update

#### Businesses

`backend/app/businesses/repositories/business_repository.py:42-44` loads by ID and calls
`_update_entity`, which is the same operation already performed by `BaseRepository.update`.

#### Communications

`backend/app/communications/repositories/correspondence_repository.py:105-113` manually loops over
fields, checks `hasattr`, assigns values, and flushes. This recreates generic update mechanics with
a behavioral difference: unknown field names are silently skipped.

The request schema currently rejects unknown external fields, but internal callers can still pass
a misspelled field and receive a successful partial/no-op update.

#### Notes

`backend/app/notes/repositories/note_entity_note_repository.py:50-57` recreates update mechanics for
one field. This may be an intentional allowlist, but the method is named the generic `update`
rather than `update_note`, so its narrower behavior is not visible in the API.

### Local reimplementations of soft delete

| Domain | Evidence | Difference from base |
| --- | --- | --- |
| Binders | `backend/app/binders/repositories/binder_repository.py:594-602` | Queries directly without the base soft-delete filter; otherwise recreates timestamp, actor, flush |
| Businesses | `backend/app/businesses/repositories/business_repository.py:46-52` | Recreates the base method using `utcnow()` |
| Clients | `backend/app/clients/repositories/client_record_repository.py:67-74` | Queries directly, requires non-null `deleted_by` |
| Communications | `backend/app/communications/repositories/correspondence_repository.py:115-122` | Uses `utcnow_aware()` |
| Notes | `backend/app/notes/repositories/note_entity_note_repository.py:59-66` | Uses `utcnow_aware()` |

The timezone-aware overrides may be necessary because their columns have different timezone
semantics. If so, that difference should be implemented through an explicit base capability or
clearly documented override, not by copying the whole operation.

The binder/business/client copies require model-column verification before removal: inherited
`BaseRepository.soft_delete` uses `utcnow()` and conditionally assigns `deleted_by`; callers may
currently depend on the stricter or direct-query behavior.

### `create` overrides reviewed

Many repositories declare `create`, but most accept a typed/domain-specific argument list, set
defaults, instantiate multiple fields deliberately, or create associated entities. They are not
classified as redundant merely because `BaseRepository.create` exists.

The annual-report root repository's `create(**kwargs)` at
`backend/app/annual_reports/repositories/annual_report_report_repository.py:34-35` is a direct
`build_and_add` delegation and is a likely additional redundant override. It remains listed
separately because its public signature and mixin composition should be checked before removal.

### Risk

- Base fixes do not reach domains that copied the implementation.
- Soft-delete timestamp awareness varies by repository.
- Silent `hasattr` filtering can hide internal programming errors.
- A method named like the generic base operation can have narrower, undocumented behavior.
- Pure delegation adds navigation noise and makes the domain appear to own a rule it does not own.

### Recommended ownership boundary

Remove pure delegations and inherit the base behavior directly. For genuine differences:

- give narrow operations narrow names;
- move reusable timezone/column behavior into an explicit base hook;
- keep an override only when its contract is documented and tested as different;
- do not retain wrappers solely to preserve an old call shape.

## R16 — Generic source-link integrity

### Shared concept

Tasks, reminders, and work-queue items refer to a source entity through a polymorphic pair:

```text
source_domain + source_id
```

`WorkQueueSourceType` and `normalize_source_domain` already provide a common vocabulary:

`backend/app/common/source_types.py:8-24`.

A valid link must be atomic:

- both fields are absent for an unlinked/manual entity; or
- both fields are present;
- the domain value is supported;
- the referenced source exists;
- when client context is derived from the source, that source is authoritative.

### Tasks implementation

Task creation calls `_validate_source` before persistence:

`backend/app/tasks/services/task_service.py:114-115,591-603`.

It enforces:

- both-null is allowed;
- a partial pair is rejected;
- unsupported domains are rejected;
- a missing source entity is rejected.

Task update adds another local implementation of pair atomicity:

`backend/app/tasks/services/task_service.py:351-378`.

It requires both fields in the same PATCH operation, allows clearing only with two explicit nulls,
revalidates the source, and recomputes `client_record_id`.

### Reminder contract gap

`ReminderCreateRequest` exposes the same two fields independently:

`backend/app/reminders/schemas/reminder.py:9-16`.

`ReminderService.create_from_request` persists both values without:

- pair validation;
- domain normalization;
- source existence validation;
- client ownership resolution.

Evidence:

`backend/app/reminders/services/reminder_service.py:62-78`.

The database model also has two independent nullable columns and no pair check constraint:

`backend/app/reminders/models/reminder.py:40-53`.

Later reminder reads attempt to normalize the domain only if both fields happen to be present:

`backend/app/reminders/services/reminder_service.py:166-191`.

The executor separately copies whichever individual values are present into task/action payloads:

`backend/app/reminders/services/reminder_executor_service.py:148-151`.

### Behavioral consequence

The system can persist:

- `source_domain` without `source_id`;
- `source_id` without `source_domain`;
- an unsupported source string;
- a link to a nonexistent entity.

Those states are impossible through the corresponding task creation path but valid through the
reminder path. Work-queue linking and client-name resolution then degrade silently instead of
rejecting the invalid relationship at write time.

### Risk

- Orphaned reminders cannot resolve client context.
- Reminder execution can create tasks or actions with malformed source metadata.
- Deduplication by `(source_domain, source_id)` becomes unreliable.
- A renamed source domain can remain stored as an arbitrary string.
- There is no database invariant protecting non-API writers.

### Recommended ownership boundary

Create one common typed source-link value or pure validator using `WorkQueueSourceType`. Tasks and
reminders should call it at creation and update boundaries. Source-existence and client-resolution
queries can remain in a dedicated repository, but the pair invariant should not be redefined per
consumer.

Consider a database check constraint enforcing both-null or both-non-null after existing data has
been audited.

## R17 — Required non-blank text

### Canonical implementation

`NonBlankStr` is the shared schema type for business-required text:

```python
Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]
```

Evidence:

`backend/app/core/api_types.py:35-38`.

It strips surrounding whitespace and rejects both empty and whitespace-only strings.

It is already used by clients, users, authority contacts, communications, and notes.

### Canonical bypasses

#### Business names

Two create schemas independently define the same `normalize_business_name` validator:

- `backend/app/businesses/schemas/business_schemas.py:12-31`;
- `backend/app/businesses/schemas/business_schemas.py:34-50`.

Both strip, reject blank text, and return the normalized string. The update schema uses a separate
inline `StringConstraints` definition at lines 53-59.

The same semantic field therefore has three local declarations.

#### Advance-payment delete reasons

`backend/app/advance_payments/schemas/advance_payment.py:189-198` declares a plain string with
`min_length=1`, then adds `_strip_reason` to reject whitespace and return the trimmed value.

#### VAT correction notes

`backend/app/vat/schemas/vat_report.py:192-201` repeats the same pattern with
`validate_correction_note`.

#### Task titles

Task update correctly uses `StringConstraints(strip_whitespace=True, min_length=1)`:

`backend/app/tasks/schemas/task.py:34-38`.

Task creation uses only `Field(min_length=1)`:

`backend/app/tasks/schemas/task.py:20-21`.

Consequently, a whitespace-only title can be created but cannot be supplied through update.

### What remains domain-specific

Maximum lengths and user-facing validation messages may differ. The shared rule is trimming plus
non-blank enforcement, not one global maximum length.

### Risk

- Create and update contracts for the same field disagree.
- Whitespace-only persisted values break display, search, and audit readability.
- A future trimming change must be repeated across validators.
- Pydantic validation errors vary unnecessarily for the same primitive constraint.

### Recommended ownership boundary

Use `NonBlankStr` directly when no maximum is needed. Add a small family of constrained aliases or
compose `StringConstraints(strip_whitespace=True, min_length=1, max_length=...)` consistently when
a maximum is required.

Do not retain field validators solely to customize the Hebrew message unless the API contract
explicitly requires that exact message.

## R18 — Bulk identifier-list validation

### Shared rule being recreated incompletely

Bulk commands accept explicit entity IDs. Independent of the domain action, a valid explicit-ID
batch generally requires:

1. at least one ID;
2. every ID is positive;
3. IDs are unique, so a caller mistake is not silently deduplicated or processed twice;
4. a bounded batch size;
5. deterministic per-ID success/failure reporting.

The domains currently enforce different subsets of this contract.

### Implementations

| Domain/request | Non-empty | Positive items | Reject duplicates | Maximum size |
| --- | --- | --- | --- | --- |
| Task bulk complete/assign | Yes | Yes | No | 100 |
| Advance-payment refresh/mark-paid | Yes | No | Yes | Domain constants |
| Charge bulk action | Yes | No | No | No |
| Binder handover | Yes | No | No | No |

Evidence:

- task requests and `_validate_positive_ids`:
  `backend/app/tasks/schemas/task.py:14-17,133-149`;
- advance-payment requests:
  `backend/app/advance_payments/schemas/advance_payment.py:364-380,390-412`;
- charge bulk request:
  `backend/app/charges/schemas/charge.py:129-132`;
- binder handover request:
  `backend/app/binders/schemas/binder.py:177-186`.

Binder intake updates also accept optional lists of business, annual-report, and VAT-report IDs
without item constraints:

`backend/app/binders/schemas/binder.py:141-143`.

### Behavioral consequences

- tasks accept duplicate IDs;
- advance payments accept zero or negative IDs at schema level, then report them as missing;
- charges and binders accept duplicates, non-positive IDs, and unbounded lists;
- repeated IDs can distort processed/succeeded/failed counts depending on service implementation;
- an unbounded charge or binder request can create avoidable transaction and response load.

The advance-payment service performs a defensive deduplication in a later path despite the schema
already rejecting duplicates:

`backend/app/advance_payments/services/advance_payment_service.py:893`.

That is another sign that batch-shape policy is distributed between schema and service.

### What remains domain-specific

- maximum batch size;
- all-or-nothing versus partial success;
- authorization and ownership checks;
- allowed domain actions;
- failure DTO shape.

### Recommended ownership boundary

Define a reusable constrained positive-ID list type or validator for positivity and uniqueness.
Each domain must still supply its approved maximum size and transaction semantics explicitly.

Do not silently convert the input to a set: order and duplicate rejection are part of the caller
contract.

## R19 — Enum-string parsing

### Shared mechanism being recreated

Several services accept raw strings, construct a set of every enum `.value`, test membership,
format an error, and then construct the enum.

### Implementations

| Domain | Enum/purpose | Evidence |
| --- | --- | --- |
| Reminders | Status filter with default `SCHEDULED` | `backend/app/reminders/services/reminder_service.py:224-233` |
| Signature requests | Optional status filter | `backend/app/signature_requests/services/signature_request_service.py:244-254` |
| Signature requests | Request type on creation | `backend/app/signature_requests/services/signature_request_creation_service.py:81-86` |
| Annual reports | Status transition target | `backend/app/annual_reports/services/annual_report_status_service.py:101-106` |
| Annual reports | Filing deadline type | `backend/app/annual_reports/services/annual_report_status_service.py:237-243` |
| Annual reports | Client filing type and deadline type | `backend/app/annual_reports/services/annual_report_create_service.py:64-78` |
| Annual reports | Income source type, create and update | `backend/app/annual_reports/services/annual_report_financial_line_service.py:86-92,123-129` |
| Annual reports | Expense category, create and update | `backend/app/annual_reports/services/annual_report_financial_line_service.py:210-216,259-265` |
| Timeline | Annual-report status reconstruction | `backend/app/timeline/repositories/timeline_repository.py:46-51` |

### Behavioral differences

- missing strings mean a domain-specific default, `None`, or an error;
- invalid strings map to different `AppError` codes;
- some error messages sort allowed values and others iterate an unordered set;
- timeline constructs the enum directly, so an invalid persisted value raises raw `ValueError`;
- some API boundaries already have Pydantic enum types, while services still repeat validation for
  internal string callers.

### Risk

- Allowed-value messages can be nondeterministic when joining an unordered set.
- Raw `ValueError` can escape from read-model reconstruction.
- Create and update paths can validate the same enum differently.
- A new enum member may be accepted automatically in a path that actually requires a narrower
  capability allowlist.

### Recommended ownership boundary

Use typed enum request fields at transport boundaries where the full enum is valid. For internal
raw-string boundaries, use one generic parsing helper that accepts:

- enum type;
- missing-value policy;
- domain error factory.

Keep capability allowlists separate from enum validity. “Known enum member” must not automatically
mean “allowed transition or operation.”

## R20 — Available-action capability rules

### Shared rule being recreated

An action descriptor shown to a caller is a claim that the corresponding mutation is currently
allowed. The action builder and mutation service must therefore use the same capability rule.

At present, some action providers reconstruct service preconditions by inspecting statuses
independently.

### Charges

`get_charge_actions` defines:

- edit, issue, and cancel for `DRAFT`;
- mark-paid and cancel for `ISSUED`;
- delete for `DRAFT` or `CANCELED`;
- no actions for roles other than advisor/secretary.

Evidence:

`backend/app/actions/services/charge_actions.py:27-65`.

`BillingService` separately owns the actual mutation rules:

- issue requires `DRAFT`:
  `backend/app/charges/services/charge_billing_service.py:161-180`;
- mark-paid requires `ISSUED`:
  `backend/app/charges/services/charge_billing_service.py:193-212`;
- cancel rejects `PAID` and `CANCELED`:
  `backend/app/charges/services/charge_billing_service.py:225-245`;
- delete permits only `DRAFT` or `CANCELED`:
  `backend/app/charges/services/charge_billing_service.py:263-278`.

The action module even documents that its delete condition “mirrors” the service. That is explicit
evidence of two owners for one rule.

### VAT

`get_vat_work_item_actions` independently decides when to expose:

- materials complete;
- add invoice;
- ready for review;
- file return;
- send back.

Evidence:

`backend/app/actions/services/vat_report_actions.py:17-68`.

The actual mutation layer uses a mixture of:

- the central `VALID_TRANSITIONS` table;
- `assert_transition_allowed`;
- direct status comparisons;
- `assert_editable`;
- additional filing readiness and assignment checks.

Evidence:

- `backend/app/vat/vat_constants.py:8-25`;
- `backend/app/vat/vat_data_entry_common.py:32-48`;
- `backend/app/vat/services/vat_data_entry_status_service.py:20-43,58-86`;
- `backend/app/vat/services/vat_filing_service.py:61-89`;
- `backend/app/vat/services/vat_data_entry_invoices_service.py:52-76`.

The action builder exposes `file_vat_return` from `READY_FOR_REVIEW`, but it cannot express other
blocking conditions such as a missing assignee. It therefore reports “available” when the status
transition is valid but the complete operation is not executable.

### Tasks

`task_actions` enables edit, complete, and cancel only when the status string is `OPEN`, and disables
them otherwise:

`backend/app/work_queue/work_queue_actions.py:45-119`.

`TaskService` separately defines `_TERMINAL`, edit restrictions, and complete/cancel behavior:

- `backend/app/tasks/services/task_service.py:41`;
- `backend/app/tasks/services/task_service.py:321-330`;
- `backend/app/tasks/services/task_service.py:380-436`.

Delete is offered independently of task status and is currently accepted by the service. This
matches today, but only through duplicated knowledge.

### Binder counterexample

Binder action generation correctly delegates to the lifecycle service:

`backend/app/actions/services/binder_actions.py:7-25`.

It demonstrates the desired ownership shape: action presentation consumes domain-owned capability
logic rather than restating it.

### Risk

- The UI can offer an action that the service rejects.
- A newly introduced service prerequisite is invisible to the action builder.
- Role restrictions can differ between endpoint dependencies and response actions.
- A status transition change requires edits in both mutation and read/response paths.
- Disabled reasons can describe only status, not the actual blocking invariant.

### Recommended ownership boundary

Each mutation domain should own named capability predicates or a capability result carrying
`allowed` and `reason`. Action builders should translate that result into an
`ActionDescriptor`. Mutation services must still recheck the predicate under the write lock.

Do not make action descriptors themselves the authorization boundary.

## R21 — Final and terminal status classification

### Shared rule being recreated

Read models repeatedly need to answer whether a source entity is:

- terminal and immutable;
- complete for a specific dashboard;
- no longer actionable in the work queue;
- eligible for automatic cancellation.

Those meanings are related but not identical. The code frequently uses broad names such as
`terminal`, `done_statuses`, `FINAL_STATUSES`, or `is_final` while rebuilding sets locally.

### Annual reports

The transition table makes only `CLOSED` and `CANCELED` structurally terminal because they have no
outgoing transitions. `SUBMITTED` can transition back to `IN_PREPARATION` or forward to `CLOSED`:

`backend/app/annual_reports/annual_report_constants.py:25-45`.

Other consumers independently classify `SUBMITTED` as final:

- dashboard final statuses:
  `backend/app/annual_reports/repositories/annual_report_report_lifecycle_repository.py:13-19`;
- work-queue due exclusion:
  `backend/app/annual_reports/repositories/annual_report_report_repository.py:40-54`;
- cancellation exclusion:
  `backend/app/annual_reports/repositories/annual_report_report_repository.py:242-253`;
- generic work-queue source state:
  `backend/app/work_queue/work_queue_source_lookup.py:93-110`.

This may be correct for those contexts, but `SUBMITTED` is “complete for dashboard/work queue,” not
terminal in the state machine. The duplicated broad terminology hides that distinction.

### VAT

`VALID_TRANSITIONS` contains `FILED` as terminal and does not contain a `CANCELED` key:

`backend/app/vat/vat_constants.py:8-25`.

The work queue treats both `FILED` and `CANCELED` as final:

`backend/app/work_queue/work_queue_source_lookup.py:74-90`.

VAT editability checks only `FILED`:

`backend/app/vat/vat_data_entry_common.py:32-35`.

Therefore a canceled item is “final” for work-queue linking but is not protected by the generic
`assert_editable` rule. Whether canceled items remain editable depends on how they can reach the
mutation path.

### Other work-queue source classifications

`work_queue_source_lookup` locally defines:

- advance payment final iff `PAID`;
- charge final iff `PAID` or `CANCELED`;
- binder final iff `HANDED_OVER`.

Evidence:

`backend/app/work_queue/work_queue_source_lookup.py:113-155`.

These classifications are not imported from the source domains. A new terminal or archived state
will therefore remain actionable in linked-task reconciliation until this read-model module is
updated.

### Risk

- “Final” can be mistaken for “immutable.”
- Work-queue cleanup and domain mutations can disagree about a source's lifecycle.
- Client-close cascades can skip or cancel different sets from dashboards.
- New enum members fail open in locally maintained exclusion lists.
- Tests tend to verify each consumer independently instead of guaranteeing cross-consumer parity.

### Recommended ownership boundary

Source domains should expose context-specific predicates or named sets:

- `is_transition_terminal`;
- `is_work_complete`;
- `is_work_queue_final`;
- `is_cancelable_on_client_close`.

Consumers should import the exact capability they need. Do not force every context into one global
`FINAL_STATUSES` set.

## R22 — User-search normalization

### Shared rule being recreated

Several backend features accept a free-text search value and decide:

- whether to trim whitespace;
- how to perform case-insensitive matching;
- which fields are stringified;
- whether numeric input is identifier-only or free text;
- how Unicode case normalization works.

### Implementations

#### Work queue

Work queue trims and applies Unicode-aware `casefold()` to both query and materialized values:

`backend/app/work_queue/services/work_queue_service.py:88-110`.

It searches a broad concatenation of source and linked-task fields.

#### Timeline

Timeline trims and applies `lower()`, then searches description, selected IDs, and stringified
metadata values:

`backend/app/timeline/services/timeline_service.py:128-140`.

Unlike `casefold()`, `lower()` is not the strongest Unicode caseless normalization. Metadata
containers are stringified shallowly and inherit Python representation details.

#### Global search

Global search has an explicit term classifier:

`backend/app/search/search_term_parser.py:18-78`.

It treats:

- period-shaped values as both period and text capabilities;
- bare integers as public identifiers, never free text;
- oversized integers specially;
- plausible years as a separate capability.

Repositories then trim again and rely on SQL `ILIKE`, for example:

`backend/app/search/repositories/search_result_repository.py:30-74,86-104`.

#### Domain SQL list searches

Client and business repositories independently construct `%term%`/`ILIKE` filters and trim at
different points:

- `backend/app/clients/repositories/client_record_repository.py:190-210,262-275,311-323`;
- `backend/app/businesses/repositories/business_repository.py:111-121`.

### Behavioral consequences

The same user input can behave differently by screen:

- a bare number searches free-text task/work-queue fields but not global-search free text;
- timeline uses `lower`, work queue uses `casefold`, and SQL uses database collation/`ILIKE`;
- period aliases such as `3/2026` are normalized only by global search;
- leading/trailing spaces are trimmed in multiple layers, with no single transport rule;
- metadata search can match Python string representations that global SQL search never exposes.

### What is domain-specific

The set of searchable fields and whether numeric identifiers are meaningful belong to each
feature. The shared concern is normalization and classification vocabulary, not forcing every
screen to search the same entities.

### Recommended ownership boundary

Reuse the global search parser or extract a smaller common normalized-term primitive. Each feature
should then explicitly choose capabilities such as text, integer, tax year, or period instead of
classifying the raw string independently.

Database and in-memory search cannot be made byte-for-byte identical without collation decisions;
tests should document intentional differences.

## Reviewed candidates that are not classified as shared-rule duplication

### Domain status machines

Annual reports and VAT both expose `VALID_TRANSITIONS`, but the allowed states and transitions are
domain behavior. They should remain separately owned. Sharing a generic transition engine may be
useful mechanically, but sharing the transition map would erase domain ownership.

Evidence:

- `backend/app/annual_reports/annual_report_constants.py:25`;
- `backend/app/vat/vat_constants.py:8`.

### Pagination defaults

Many routes use a literal default of 20 while `app.core.pagination.DEFAULT_PAGE_SIZE` is 50.
Different list shapes may legitimately choose different payload sizes, so this audit does not claim
that every default must be one global value.

The shared rules that are binding are:

- maximum page size from `MAX_PAGE_SIZE`;
- SQL pagination through `BaseRepository.apply_pagination`;
- in-memory pagination through `paginate_sequence`.

No additional hand-written SQL or sequence pagination was found outside the canonical helpers.

### Domain-specific field normalization

Trimming optional VAT invoice strings, business-name normalization, period parsing, and correction
note validation are not grouped merely because they use `.strip()`. Their accepted input and error
semantics belong to different contracts.

### Entity-specific audit snapshots

Audit snapshot and metadata dictionaries naturally differ by entity. Only actor inference is
classified as duplicated cross-domain policy.

### Notification eligibility

Notification trigger rules reference binders, annual reports, VAT, charges, invoices, and signature
requests. Those checks are orchestration owned by notifications, not automatically duplication of
the source domains. Only the generic client-active substitute in R3 is flagged.

## Consolidation order

This is a recommended risk order, not an approved implementation plan.

1. **R3 client eligibility** — resolve behavioral inconsistencies and confirm intentional
   notification exceptions.
2. **R4 ownership validation** — close IDOR-sensitive create/update differences first.
3. **R11 tax-rules access** — eliminate the hidden fallback and competing financial types.
4. **R7 PATCH contract** — align the charge request with the already-binding shared contract.
5. **R8 clocks** — move business decisions to the canonical clock and separate export-local time.
6. **R13 soft-delete lifecycle** — define audit and missing/deleted-state conventions before
   extracting mechanics.
7. **R2 actor attribution** — establish one audit-owned actor context.
8. **R10 audit serialization** — remove caller-side partial normalization.
9. **R12 cadence validation** — centralize the formula while retaining validation at each boundary.
10. **R14 client identity** — route shared display fields through the existing repository.
11. **R1 due-date snapshots** — consolidate the exact event duplication while preserving the
   advance-payment legacy initializer.
12. **R6 email normalization** — consolidate current behavior before considering any semantic
   normalization change.
13. **R9 explicit-null rejection** — extract the mechanism while leaving field declarations local.
14. **R15 repository overrides** — remove pure delegations first; verify timestamp-column
   semantics before consolidating copied soft-delete implementations.
15. **R16 source links** — make the pair atomic in reminders and share source resolution with
   tasks/work queue.
16. **R17 non-blank text** — replace local validators and align task create/update behavior.
17. **R18 bulk IDs** — establish positive, unique, bounded list primitives before changing command
   semantics.
18. **R19 enum parsing** — type transport inputs and centralize only the parsing mechanism.
19. **R20 capabilities** — make actions consume domain predicates and recheck them under lock.
20. **R21 lifecycle classification** — replace broad “final” sets with context-specific domain
   predicates.
21. **R22 search normalization** — share term capabilities while preserving feature-specific fields.
22. **R5 lookups** — standardize only after defining owner-scoped, locked, token, and
   include-deleted variants.

## Verification requirements for future remediation

Any implementation derived from this audit must be verified against the risk-appropriate rules in
`docs/workflow/verification.md`. At minimum:

- preserve or deliberately update the owning domain documentation when behavior changes;
- run backend lint, type checking, and targeted/full tests as required by
  `docs/backend/testing.md`;
- verify OpenAPI when PATCH validation or documented errors change;
- test wrong-owner IDs on create, read, update, and delete paths;
- test closed, frozen, active, and unknown/future client statuses;
- test clock-sensitive behavior around UTC/Israel midnight;
- test system, user, and external actor audit rows;
- verify insert and update behavior for both VAT and advance-payment due-date snapshots.

## Audit limitations

- This is a static audit. It does not prove that every duplicated branch is reachable through the
  public API.
- Dynamic policy assembled from configuration or database data was not treated as duplication
  unless a local code rule was also present.
- Frontend duplication is out of scope.
- Tests were inspected through implementation references only; this audit did not execute the
  backend test suite because no runtime code was changed.
- “Complete” means complete for the 22 defined rule families and the verified `backend/app` snapshot,
  not a guarantee that arbitrary future semantic similarities can be found mechanically.
