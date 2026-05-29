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
| F-001 | High (IDOR) | annual-reports | Income/expense line update & delete check only that the report exists, not that the line belongs to it. A valid `line_id` from another report can be mutated/deleted via an unrelated `report_id`. | `app/annual_reports/services/financial_service.py:172-183, 268-283` | open |
| F-002 | Medium | annual-reports | Transition to `pending_client` silently skips signature creation when client/business not found, while the status change still succeeds. | `app/annual_reports/services/status_signature_helper.py:42` | open |
| F-003 | Low | annual-reports | Legacy docs expect tax-calendar/reminder sync on status transitions; `transition_status` does status/history/audit/signatures only — no sync. | `app/annual_reports/services/status_service.py:82` | open |
| F-004 | Low / design | annual-reports | VAT auto-populate aggregates by `client_record_id`+`tax_year`, not per business — risky for multi-business clients if business-specific import is expected. | `app/annual_reports/services/vat_import_service.py:119` | open |
| F-005 | Medium | advance-payments | `timing_status` / `paid_late` compute overdue/late from legacy `due_date`, not `due_date_effective` — contradicts INV-05. Overridden effective dates produce wrong signals. | `app/advance_payments/schemas/advance_payment.py:49-51, 57-60` | open |
| F-006 | Medium | businesses | `constants.py` imports `EntityType` from `models/business.py`, which does not define it — `ImportError` whenever the module loads. Should import from the legal entities model. | `app/businesses/constants.py:1` | open |
| F-007 | Medium | vat-reports | Filing does not enforce `assigned_to` before moving to `filed`, contradicting INV-09. | `app/vat_reports/services/filing.py:58-99` | open |
| F-008 | High (IDOR) | vat-reports | Invoice update passes `business_activity_id` through without the ownership check that create enforces — allows attaching an activity from another legal entity. | `app/vat_reports/services/data_entry_invoice_update.py:81-121` (cf. create `data_entry_invoices.py:88-94`) | open |
| F-009 | Low | vat-reports | Several VAT service errors bypass the `DOMAIN.REASON` format (`AMENDED_ITEM_NOT_FOUND`, `MISSING_FINAL_AMOUNT`, `INVALID_NET_AMOUNT`, etc.). | `app/vat_reports/services/filing.py:31-40, 72-73`, `data_entry_invoice_update.py:76-77` | open |
| F-010 | Low | vat-reports | `is_amendment=True` accepted without `amends_item_id` — amendment validation only runs when the link is present. | `app/vat_reports/services/filing.py:55-65` | open |
| F-011 | Low / gap | binders | `BinderIntakeEditService.edit_intake` is fully implemented with cross-client FK ownership validation but no HTTP endpoint exists in openapi.json — capability inaccessible via API. | `app/binders/services/binder_intake_edit_service.py:57` | open |
| F-012 | Low | tasks | `GET /api/v1/tasks` declares `assigned_role` and `source_domain` as plain `str` instead of `UserRole`/validated source-domain enum — invalid values bypass validation and silently return empty results. | `app/tasks/api/routes.py:31-32`, `app/tasks/services/task_service.py:50-52` | open |
| F-013 | Low | charge | `ChargeListResponse.stats` is computed on a broader slice than the applied list filters — UI counters disagree with the visible rows. | `app/charge/services/charge_query_service.py` | open |
| F-014 | Low | charge | `POST /api/v1/charges/bulk-action` openapi declares `X-Idempotency-Key` as `required=false`, but runtime `require_idempotency_key` dependency returns 400 when missing. Spec/runtime mismatch. | `app/charge/api/charge.py:134` (openapi.json bulk-action params) | open |
| F-015 | High (broken feature) | reminders | `ReminderExecutorService._execute` unconditionally sets status to `FAILED` with an "not implemented" reason — no action type is actually executed. All reminders are no-ops. | `app/reminders/services/reminder_executor_service.py:46-60` | open |
| F-016 | High (broken feature) | reminders | No background job calls `ReminderExecutorService.fire_due()` — even if execution were implemented, nothing would trigger it. Scheduled reminders sit forever. | `app/core/background_jobs.py`, `app/lifespan.py` (absence) | open |
| F-017 | Low | notes | `entity_note_service.list_notes`/`add_note` don't check the parent `ClientRecord` exists — writes/reads against a non-existent client succeed silently instead of 404. | `app/notes/services/entity_note_service.py:38-67` | open |
| F-018 | Low | notes | `_assert_business_belongs_to_client` raises `BUSINESS.NOT_FOUND` with a business-flavored Hebrew message even when the client (not the business) is missing. Wrong code + misleading message. | `app/notes/services/business_note_service.py:85-88` | open |
| F-019 | Low (doc-only) | correspondence | `app/correspondence/README.md` documents `CORRESPONDENCE.FORBIDDEN_BUSINESS` as raised; code never raises it (business ownership uses `BUSINESS.NOT_FOUND` via `business_guards.py`). Stale README. | `app/correspondence/README.md:127` | open |
| F-020 | Medium | work-queue | Advance-payment rows in the queue use legacy `AdvancePayment.due_date` instead of `due_date_effective` — same INV-05 violation as F-005, just in the queue source builder. | `app/work_queue/services/billing_items.py:33,47` | open |
| F-021 | Low / design | work-queue | When `business_id` filter is set, all system source items (VAT/advance/binders/charges) are skipped entirely, so the queue scoped to a business shows tasks only. Filter-by-business silently hides system work. | `app/work_queue/services/work_queue_service.py:287-288` | open |
| F-022 | High (IDOR) | authority-contact | `GET/PATCH/DELETE /api/v1/clients/authority-contacts/{contact_id}` has no client scope in URL or service — `get_contact`/`update_contact`/`delete_contact` resolve by bare `contact_id` with zero ownership check. Any ADVISOR/SECRETARY can read/update any contact system-wide; any ADVISOR can delete. | `app/authority_contact/services/authority_contact_service.py:45,78,84` | open |
| F-023 | Low (doc-only) | authority-contact | Model docstring documents `AuthorityContactLink` many-to-many model and a `business_id` column as design fact — neither exists in code. | `app/authority_contact/models/authority_contact.py` | open |
| F-024 | Medium | notifications | `POST /preview` and `POST /send` are router-level guarded with `ADVISOR or SECRETARY` — no advisor-only route guard. Manual notifications can be sent by SECRETARY despite the historical advisor-only responsibility split. | `app/notification/api/notifications.py:28-32,86-137` | open |
| F-025 | Low | notifications | `GET /api/v1/notifications` runtime accepts only `page_size ∈ {25, 50}`, but openapi advertises `1..50`. Spec/runtime mismatch. | `app/notification/api/notifications.py:26,46-53` (openapi.json:8059-8069) | open |
| F-026 | Low | permanent-documents | `upload_document` raises `PERMANENT_DOCUMENTS.CLIENT_NOT_FOUND` when the business (not the client) is missing — wrong code for the condition. | `app/permanent_documents/services/permanent_document_service.py:119` | open |
| F-027 | Low / design | permanent-documents | `CLIENT_SCOPE_TYPES` defined in model but never imported or enforced in service — types meant to be client-scoped (`id_copy`, `power_of_attorney`, `engagement_agreement`) can be uploaded with scope=BUSINESS. | `models/permanent_document.py:68` vs `services/permanent_document_service.py:126` | open |
| F-028 | High (IDOR) | permanent-documents | `DELETE /api/v1/documents/{document_id}` and `PUT /{document_id}/replace` resolve by bare `document_id` with no caller-side ownership check — any ADVISOR can delete/replace any document system-wide. | `app/permanent_documents/api/permanent_documents.py:104-133`, `services/permanent_document_service.py:257-306` | open |
| F-029 | Low | signature-requests | `GET /api/v1/signature-requests/{request_id}` raises raw `HTTPException` instead of `NotFoundError("…", "SIGNATURE_REQUEST.NOT_FOUND")` — bypasses the error envelope. | `app/signature_requests/api/routes_advisor.py:81` | open |
| F-030 | Medium (IDOR) | signature-requests | `cancel_request` fetches by bare `request_id` with no client ownership check — any ADVISOR can cancel any pending request system-wide. | `app/signature_requests/services/admin_actions.py:21-35`, `api/routes_advisor.py:89-103` | open |
| F-031 | Low (dead artifact) | signature-requests | Orphaned bytecode `services/__pycache__/send_request.cpython-314.pyc` with no source `.py`. | `app/signature_requests/services/__pycache__/` | open |
| F-032 | Low (doc-only) | search | `app/search/README.md` references non-existent `search_filters.py`, `test_search_filters.py`, and a `DocumentSearchResult.status` field that the schema does not define. | `app/search/README.md:34-37,68-70,129-137` | open |
| F-033 | Low (doc-only) | dashboard | README references `dashboard_extended_service.py`, `dashboard_extended_builders.py`, and three test files — none exist on disk. | `app/dashboard/README.md` (Implementation references) | open |
| F-034 | Low | dashboard | `quick_actions` schema field always returns `[]` — `_build_quick_actions` is a stub. Dead schema contract. | `app/dashboard/services/dashboard_overview_service.py:98-99` | open |
| F-035 | Low | dashboard | `AdvisorTodayService.build` unconditionally returns `{"deadline_items": []}` — feature not implemented. | `app/dashboard/services/advisor_today_service.py:12-13` | open |
| F-036 | Low (doc-only) | dashboard | `DASHBOARD.LIMIT_EXCEEDED` documented in README; source service does not exist → code unreachable. | `app/dashboard/README.md` (Error Envelope) | open |
| F-037 | Low | dashboard | `reports_not_started` can go negative when report counts exceed active-business count — no clamp. | `app/dashboard/services/dashboard_tax_service.py:45` | open |
| F-038 | Low / design | reports | `VatComplianceReportItemResponse` exposes `reporting_frequency` always set to the same value as `period_type` — redundant field, inflates payload, misleads consumers. | `app/reports/services/vat_compliance_report.py:55-56`, `app/reports/schemas.py:15-16` | open |
| F-039 | Low (doc/code divergence) | timeline | README claims `notification_sent` events are excluded as noisy, but service emits both `notification_sent` and `notification_failed`. Either drop or update the claim. | `app/timeline/services/timeline_service.py:129-141` vs `app/timeline/README.md:113` | open |
| F-040 | Low (temporal accuracy) | timeline | `decline_reason` is read from the live `SignatureRequest` row rather than the audit-event snapshot — later mutation of the row rewrites historical timeline. Comment in code already acknowledges this. | `app/timeline/services/timeline_client_builders.py:70-73` | open |

## Notes

- **F-001 is the priority.** Cross-report mutation via unchecked `line_id`. Suggested fix: scope repository update/delete by both `line_id` and `annual_report_id`, matching the annex service ownership check at `app/annual_reports/services/annex_service.py:87`.
- New findings from later domain runs append here. Keep each domain doc's `## Known issues` as the source detail; this table is the index.
