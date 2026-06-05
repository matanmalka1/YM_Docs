## Scope
This file owns only:
- A central index of code findings (security + correctness) surfaced during domain-docs authoring.

This file must not contain:
- The fixes themselves (these are code changes, tracked separately).
- Domain documentation (lives in `docs/domains/*`).

Source of truth: reference

# Security & Code Findings

Central tracker for code issues found while writing canonical domain docs. Each domain doc has its own `## Known issues` section; this file aggregates them for prioritization. These are **code** problems — docs-only authoring does not fix them.

Status legend: `open` / `in-progress` / `fixed`.

## Findings

| ID | Severity | Domain | Issue | Location | Status |
|----|----------|--------|-------|----------|--------|
| F-001 | High (IDOR) | annual-reports | Income/expense line update & delete check only that the report exists, not that the line belongs to it. A valid `line_id` from another report can be mutated/deleted via an unrelated `report_id`. | `app/annual_reports/services/financial_service.py:172-183, 268-283` | fixed |
| F-002 | Medium | annual-reports | Transition to `pending_client` silently skips signature creation when client/business not found, while the status change still succeeds. | `app/annual_reports/services/status_signature_helper.py:42` | fixed |
| F-003 | Low | annual-reports | Legacy docs expected tax-calendar/reminder sync on status transitions. Not a bug: `TaxCalendarEntry` has no status field; grouped calendar reads `AnnualReport.status` live via JOIN (`tax_calendar/services/grouped_service.py:214-221`, `repositories/grouped_repository.py:141-166`). Reminders module has no coupling to annual-reports. Legacy sync expectation is structurally retired. | `app/annual_reports/services/status_service.py:82` | resolved — not a bug |
| F-004 | Low / design | annual-reports | VAT auto-populate aggregates by `client_record_id`+`tax_year`, not per business — risky for multi-business clients if business-specific import is expected. | `app/annual_reports/services/vat_import_service.py:119` | accepted-design |
| F-005 | Medium | advance-payments | `timing_status` / `paid_late` compute overdue/late from legacy `due_date`, not `due_date_effective` — contradicts INV-05. Overridden effective dates produce wrong signals. | `app/advance_payments/schemas/advance_payment.py:49-51, 57-60` | fixed |
| F-006 | Medium | businesses | `constants.py` imported `EntityType` from `models/business.py`, which does not define it — `ImportError` whenever the module loads. Fixed to import the enum source of truth directly from `app.common.enums`. | `app/businesses/constants.py:1` | fixed |
| F-007 | Medium | vat-reports | Filing does not enforce `assigned_to` before moving to `filed`, contradicting INV-09. | `app/vat_reports/services/filing.py:58-99` | fixed |
| F-008 | High (IDOR) | vat-reports | Invoice update passes `business_activity_id` through without the ownership check that create enforces — allows attaching an activity from another legal entity. | `app/vat_reports/services/data_entry_invoice_update.py:81-121` (cf. create `data_entry_invoices.py:88-94`) | fixed |
| F-009 | Low | vat-reports | Several VAT service errors bypass the `DOMAIN.REASON` format (`AMENDED_ITEM_NOT_FOUND`, `MISSING_FINAL_AMOUNT`, `INVALID_NET_AMOUNT`, etc.). | `app/vat_reports/services/filing.py:31-40, 72-73`, `data_entry_invoice_update.py:76-77` | fixed |
| F-010 | Low | vat-reports | `is_amendment=True` accepted without `amends_item_id` — amendment validation only runs when the link is present. | `app/vat_reports/services/filing.py:55-65` | fixed |
| F-011 | Low / gap | binders | `BinderIntakeEditService.edit_intake` is fully implemented with cross-client FK ownership validation but no HTTP endpoint exists in openapi.json — capability inaccessible via API. | `app/binders/services/binder_intake_edit_service.py:57` | fixed |
| F-012 | Low | tasks | `GET /api/v1/tasks` declares `assigned_role` and `source_domain` as plain `str` instead of `UserRole`/validated source-domain enum — invalid values bypass validation and silently return empty results. | `app/tasks/api/routes.py:31-32`, `app/tasks/services/task_service.py:50-52` | fixed |
| F-013 | Low | charge | `ChargeListResponse.stats` is computed on a broader slice than the applied list filters — UI counters disagree with the visible rows. | `app/charge/services/charge_query_service.py` | fixed |
| F-014 | Low | charge | `POST /api/v1/charges/bulk-action` openapi declares `X-Idempotency-Key` as `required=false`, but runtime `require_idempotency_key` dependency returns 400 when missing. Spec/runtime mismatch. | `app/charge/api/charge.py:134` (openapi.json bulk-action params) | fixed |
| F-015 | High (broken feature) | reminders | `ReminderExecutorService._execute` unconditionally sets status to `FAILED` with an "not implemented" reason — no action type is actually executed. All reminders are no-ops. | `app/reminders/services/reminder_executor_service.py:46-60` | open |
| F-016 | High (broken feature) | reminders | No background job calls `ReminderExecutorService.fire_due()` — even if execution were implemented, nothing would trigger it. Scheduled reminders sit forever. | `app/core/background_jobs.py`, `app/lifespan.py` (absence) | open |
| F-017 | Low | notes | `entity_note_service.list_notes`/`add_note` don't check the parent `ClientRecord` exists — writes/reads against a non-existent client succeed silently instead of 404. | `app/notes/services/entity_note_service.py:38-67` | fixed |
| F-018 | Low | notes | `_assert_business_belongs_to_client` raises `BUSINESS.NOT_FOUND` with a business-flavored Hebrew message even when the client (not the business) is missing. Wrong code + misleading message. | `app/notes/services/business_note_service.py:85-88` | fixed |
| F-019 | Low (doc-only) | correspondence | `app/correspondence/README.md` documents `CORRESPONDENCE.FORBIDDEN_BUSINESS` as raised; code never raises it (business ownership uses `BUSINESS.NOT_FOUND` via `business_guards.py`). Stale README. | `app/correspondence/README.md:127` | fixed |
| F-020 | Medium | work-queue | Advance-payment rows in the queue use legacy `AdvancePayment.due_date` instead of `due_date_effective` — same INV-05 violation as F-005, just in the queue source builder. | `app/work_queue/services/billing_items.py:33,47` | fixed |
| F-021 | Low / design | work-queue | When `business_id` filter is set, task merge was skipped entirely — tasks linked to charge rows were silently dropped from business-scoped queue. Root cause: `business_id is not None` short-circuit in `_merge_tasks`. Note: VAT/annual/advance sources are correctly excluded by design; only charge rows appear under `business_id`. Standalone tasks (no business scope) are also correctly excluded. | `app/work_queue/services/work_queue_service.py:346` | fixed |
| F-022 | High (IDOR) | authority-contact | `GET/PATCH/DELETE /api/v1/clients/authority-contacts/{contact_id}` has no client scope in URL or service — `get_contact`/`update_contact`/`delete_contact` resolve by bare `contact_id` with zero ownership check. Any ADVISOR/SECRETARY can read/update any contact system-wide; any ADVISOR can delete. | `app/authority_contact/services/authority_contact_service.py:45,78,84` | fixed |
| F-023 | Low (doc-only) | authority-contact | Model docstring documents `AuthorityContactLink` many-to-many model and a `business_id` column as design fact — neither exists in code. | `app/authority_contact/models/authority_contact.py` | fixed |
| F-024 | Medium | notifications | Resolved by product decision: notification preview/send intentionally share the router-level `ADVISOR or SECRETARY` permission model. No advisor-only route guard is required. | `app/notification/api/notifications.py` | fixed |
| F-025 | Low | notifications | `GET /api/v1/notifications` now publishes `page_size ∈ {25, 50}` in OpenAPI, matching runtime validation. | `app/notification/api/notifications.py` (openapi.json) | fixed |
| F-026 | Low | permanent-documents | `upload_document` raises `PERMANENT_DOCUMENTS.CLIENT_NOT_FOUND` when the business (not the client) is missing — wrong code for the condition. | `app/permanent_documents/services/permanent_document_service.py:119` | fixed |
| F-027 | Low / design | permanent-documents | `CLIENT_SCOPE_TYPES` defined in model but never imported or enforced in service — types meant to be client-scoped (`id_copy`, `power_of_attorney`, `engagement_agreement`) can be uploaded with scope=BUSINESS. | `models/permanent_document.py:68` vs `services/permanent_document_service.py:126` | fixed |
| F-028 | High (IDOR) | permanent-documents | `DELETE /api/v1/documents/{document_id}` and `PUT /{document_id}/replace` resolve by bare `document_id` with no caller-side ownership check — any ADVISOR can delete/replace any document system-wide. | `app/permanent_documents/api/permanent_documents.py:104-133`, `services/permanent_document_service.py:257-306` | fixed |
| F-042 | Medium (IDOR) | permanent-documents | `GET /api/v1/documents/{document_id}/download-url` resolves by bare `document_id` with no client ownership check — any ADVISOR/SECRETARY can obtain a presigned URL for any document by guessing `document_id`. Same pattern as F-028 but on the read path. | `app/permanent_documents/api/permanent_documents.py:93-101`, `services/permanent_document_service.py:194-198` | fixed |
| F-029 | Low | signature-requests | Detail lookup raised raw `HTTPException`; the global handler preserved the envelope, but emitted generic `not_found` instead of `SIGNATURE_REQUEST.NOT_FOUND`. Replaced with the domain `NotFoundError` path. | `app/signature_requests/api/routes_advisor.py`, `services/signature_request_service.py` | fixed |
| F-030 | Medium (broken ownership boundary) | signature-requests | Cancellation used bare `request_id` with no client ownership context. Replaced by client-scoped route/service/repository contract and locked pending lookup; bare cancellation APIs removed. | `app/signature_requests/api/routes_client.py`, `services/admin_actions.py`, `repositories/signature_request_crud.py` | fixed |
| F-031 | Low (dead artifact) | signature-requests | Reported orphaned `send_request.cpython-314.pyc` is absent; `__pycache__/` is already ignored. Closed as stale. | `app/signature_requests/services/__pycache__/` | fixed — stale |
| F-032 | Low (doc-only) | search | Reported stale README content is absent; `app/search/README.md` is already a thin canonical-doc pointer. Closed as stale. | `app/search/README.md` | fixed — stale |
| F-033 | Low (doc-only) | dashboard | README references `dashboard_extended_service.py`, `dashboard_extended_builders.py`, and three test files — none exist on disk. | `app/dashboard/README.md` (Implementation references) | open |
| F-034 | Low | dashboard | `quick_actions` schema field always returns `[]` — `_build_quick_actions` is a stub. Dead schema contract. | `app/dashboard/services/dashboard_overview_service.py:98-99` | open |
| F-035 | Low | dashboard | `AdvisorTodayService.build` unconditionally returns `{"deadline_items": []}` — feature not implemented. | `app/dashboard/services/advisor_today_service.py:12-13` | open |
| F-036 | Low (doc-only) | dashboard | `DASHBOARD.LIMIT_EXCEEDED` documented in README; source service does not exist → code unreachable. | `app/dashboard/README.md` (Error Envelope) | open |
| F-037 | Low | dashboard | `reports_not_started` can go negative when report counts exceed active-business count — no clamp. | `app/dashboard/services/dashboard_tax_service.py:45` | open |
| F-038 | Low / design | reports | `VatComplianceReportItemResponse` exposes `reporting_frequency` always set to the same value as `period_type` — redundant field, inflates payload, misleads consumers. | `app/reports/services/vat_compliance_report.py:55-56`, `app/reports/schemas.py:15-16` | open |
| F-039 | Low (doc/code divergence) | timeline | README claims `notification_sent` events are excluded as noisy, but service emits both `notification_sent` and `notification_failed`. Either drop or update the claim. | `app/timeline/services/timeline_service.py:129-141` vs `app/timeline/README.md:113` | open |
| F-040 | Low (temporal accuracy) | timeline | `decline_reason` is read from the live `SignatureRequest` row rather than the audit-event snapshot — later mutation of the row rewrites historical timeline. Comment in code already acknowledges this. | `app/timeline/services/timeline_client_builders.py:70-73` | open |
| F-041 | Low | users | `GET /api/v1/users/audit-logs` accepts `from`/`to` without validating `from <= to` — reversed range silently returns empty results. | `app/users/repositories/user_audit_log_repository.py:103-106` | open |

## Notes

- **F-001 fixed.** Income and expense update/delete now scope repository mutation and audit snapshots by both `line_id` and `annual_report_id`.
- **F-004 accepted-design.** VAT auto-populate is intentionally client-wide. Annual reports are client-scoped; Business is activity grouping only. Adding `business_id` filtering would create a client-level report with a business-level import — inconsistency. Decision recorded in `docs/domains/annual-reports.md` Decisions section; test coverage in `test_vat_auto_populate_aggregates_all_businesses_for_client_year`.
- New findings from later domain runs append here. Keep each domain doc's `## Known issues` as the source detail; this table is the index.
