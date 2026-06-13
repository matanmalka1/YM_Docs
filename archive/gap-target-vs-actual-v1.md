## Scope
This file owns only:
- A historical target-vs-actual domain gap analysis and build-order plan.

This file must not contain:
- Active architecture guidance or canonical domain behavior (see `docs/domains/*` and `docs/project/backend-module-map.md`). Note: this file references a `target-domains-v1.md` that no longer exists in this repo.

Source of truth: historical (non-canonical)

# Gap Analysis — Target vs Actual (v1)

> Compares `target-domains-v1.md` (planned 30 domains + actions) against
> `actual-domains-v1.md` (what `backend/app/` exposes today).
>
> Match is by **behavior**, not exact name — actual code often renames
> (`soft_delete_client()` → route `delete_client` + service `delete_client`).
>
> Legend:
> - ✅ **exists** — implemented (API endpoint and/or service function)
> - 🟡 **partial / renamed / internal** — logic exists but not as a clean public action, or named differently, or only as service (no endpoint)
> - ❌ **missing** — no code found

---

## Summary

| # | Target Domain | Actual folder | Folder state |
|---|---|---|---|
| 1 | Clients | `clients/` | full (api+svc+repo) |
| 2 | Legal Entities | `legal_entities/` | **repo only** — no api, no service |
| 3 | Businesses | `businesses/` | full |
| 4 | Contacts | `contacts/` | **empty shell** — folders exist, zero code |
| 5 | Authority Contacts | `authority_contacts/` | full |
| 6 | Notes | `notes/` | full |
| 7 | Documents | `documents/permanent_documents/` | subdomain only |
| 8 | Signature Requests | `signature_requests/` | full |
| 9 | Tasks | `tasks/` | full |
| 10 | Reminders | `reminders/` | full (named routers) |
| 11 | Alerts | `alerts/` | **service only** (one `build`) |
| 12 | Charges | `charges/` | full |
| 13 | Invoices | `invoices/` | full (thin) |
| 14 | Advance Payments | `advance_payments/` | full |
| 15 | VAT | `vat/` | full |
| 16 | Annual Reports | `annual_reports/` | full |
| 17 | National Insurance | — | **no folder** (only `ni_engine.py` inside annual_reports) |
| 18 | Tax Calendar | `tax_calendar/` | full (settings/read-heavy) |
| 19 | Binders | `binders/` | full |
| 20 | Communication | `communications/` | full (correspondence only) |
| 21 | Notifications | `notifications/` | full |
| 22 | Search | `search/` | api+svc, no repo |
| 23 | Dashboard | `dashboard/` | api+svc, no repo |
| 24 | Work Queue | `work_queue/` | api+svc, no repo |
| 25 | Actions | `actions/` | **service only**, no api |
| 26 | Reports | `reports/` | api+svc, no repo |
| 27 | Audit | `audit/` | full |
| 28 | Users | `users/` | full |
| 29 | Permissions | — | **no folder** (logic in core/deps) |
| 30 | Client Import | `clients/` | folded into clients (excel) |

**Domains entirely missing as code:** National Insurance (17), Permissions (29).
**Domains existing as empty/near-empty shells:** Contacts (4), Legal Entities (2 — no service/api), Alerts (11 — internal only), Actions (25 — no api).

---

## 1. Clients

| Target action | Status | Actual |
|---|---|---|
| create_client | ✅ | POST `/` + `create_client` svc |
| get_client | ✅ | GET `/{id}` + `get_full_client` |
| get_client_summary | 🟡 | partial via `get_client_stats` / status-card (businesses) |
| get_client_operational_status | 🟡 | partial via businesses status-card / signals |
| list_clients | ✅ | GET `/` + sidebar |
| update_client | ✅ | PATCH `/{id}` |
| soft_delete_client | ✅ | DELETE `/{id}` → `delete_client` (soft) |
| restore_client | ✅ | POST `/{id}/restore` |
| activate_client | 🟡 | no dedicated endpoint; status via `update_status` repo |
| deactivate_client | 🟡 | same — no explicit action |
| freeze_client | ❌ | no freeze action |
| unfreeze_client | ❌ | — |
| close_client | 🟡 | status-based, no endpoint |
| reopen_client | 🟡 | status-based, no endpoint |
| assign_accountant | ❌ | no ownership endpoints |
| change_accountant | ❌ | — |
| unassign_accountant | ❌ | — |
| transfer_client_ownership | ❌ | — |

> **Lifecycle + ownership are the big gap in Clients.** No freeze/close/reopen endpoints, no accountant assignment at all.

---

## 2. Legal Entities

| Target action | Status | Actual |
|---|---|---|
| create_legal_entity | 🟡 | repo `create` only (called from clients onboarding) |
| get_legal_entity | 🟡 | repo `get_by_id` only |
| list_legal_entities | ❌ | no list |
| update_legal_entity | ❌ | no service/endpoint |
| soft_delete / restore | ❌ | — |
| update_tax_id / vat_number / validate_identity | ❌ | — |
| address / phone / email | ❌ | — |
| tax profile (get + update vat/income/ni/file) | ❌ | no tax-profile endpoints |
| status (activate/deactivate/close/reopen) | ❌ | — |

> **Legal Entities is repo-only.** This is the source-of-truth domain for tax classification per `target` — and it has no service or API layer. **Largest architectural gap.**

---

## 3. Businesses

| Target action | Status | Actual |
|---|---|---|
| create_business | ✅ | POST `/` |
| get_business | ✅ | GET `/{business_id}` |
| list_businesses / list_client_businesses | ✅ | GET `/` |
| list_active_client_businesses | 🟡 | filter on list, no dedicated route |
| update_business | ✅ | PATCH `/{business_id}` |
| soft_delete_business | ✅ | DELETE → soft |
| restore_business | ✅ | POST `/{business_id}/restore` |
| open/close/reopen_business_activity | 🟡 | status via update, no explicit lifecycle endpoints |
| link/unlink/transfer business to client | ❌ | no link/transfer actions |
| set_primary_business | ❌ | — |
| update_business_name / status | 🟡 | via update_business |
| phone / email / address overrides | 🟡 | service `contact_*` exists, no dedicated endpoints |

---

## 4. Contacts

| Target action | Status |
|---|---|
| **all** (create/get/list/update/delete, client links, roles, comms) | ❌ |

> **Contacts is an empty shell.** `api/`, `services/`, `repositories/` folders exist with no code. Entire domain unimplemented.

---

## 5. Authority Contacts

| Target action | Status | Actual |
|---|---|---|
| create_authority_contact | ✅ | POST |
| get / list / list_client | ✅ | GET routes |
| update | ✅ | PATCH |
| soft_delete / restore | 🟡 | DELETE exists; no restore |
| link / unlink to client | 🟡 | implicit (always client-scoped); no explicit link endpoints |

---

## 6. Notes

| Target action | Status | Actual |
|---|---|---|
| create / get / list / update / delete | ✅ | entity_notes + business_notes routes |
| list_client / list_business / list_entity notes | ✅ | scoped routes |
| pin_note / unpin_note | ❌ | no pin support |

---

## 7. Documents

| Target action | Status | Actual |
|---|---|---|
| upload / get / list / update_metadata / soft_delete | 🟡 | only **permanent_documents** flavor |
| upload_client / list_client / list_business | ✅ | present (permanent) |
| permanent docs (create/get/list/expiring/expired/valid/expired/replace) | 🟡 | partial — upload/list/download/delete/replace exist; expiry-state actions (`mark_valid`, `mark_expired`, `list_expiring`) **missing** |
| required documents (all) | ❌ | no required-documents subdomain |
| document folders (all) | ❌ | no folders |
| classify / tag / untag | ❌ | none |
| download_url / preview_url | 🟡 | download-url yes, preview no |

> **Big gap:** general documents, required documents, folders, tagging, expiry lifecycle — all absent. Only permanent-document upload/replace exists.

---

## 8. Signature Requests

| Target action | Status | Actual |
|---|---|---|
| create / create_for_document | 🟡 | `create_signature_request` (one form) |
| get / get_status / list / list_client / list_pending / list_expired | 🟡 | get, list_client, list_pending exist; status/list_expired missing as endpoints |
| cancel | ✅ | client cancel route |
| approve / decline | ✅ | signer approve/decline |
| resend | ❌ | no resend |
| expire | 🟡 | service `expire_overdue_requests` (no endpoint) |
| get_signing_url / verify_signing_token | 🟡 | `signer_view` by token covers view; no explicit url/verify split |

---

## 9. Tasks

| Target action | Status | Actual |
|---|---|---|
| create / get / list / update / delete | ✅ | full |
| create_client_task / list_client_task | 🟡 | client scoping via filters, no dedicated routes |
| assign / reassign / unassign | ❌ | no assignment endpoints |
| complete / reopen / cancel | 🟡 | complete + cancel exist; **reopen missing** |
| snooze / mark_overdue | ❌ | none |
| list_overdue / list_due_today | 🟡 | via work_queue, not task routes |
| set_priority / update_due_date | 🟡 | via update_task |

---

## 10. Reminders

| Target action | Status | Actual |
|---|---|---|
| create / get / list / list_client | ✅ | create/list/get routes |
| update / delete | ❌ | no update or delete endpoint |
| trigger | 🟡 | service `fire_due` (scheduler, no endpoint) |
| snooze / dismiss / complete | 🟡 | only `cancel` endpoint exists |

---

## 11. Alerts

| Target action | Status |
|---|---|
| **all** (get_client_alerts, summary, sidebar counts, mark_read, dismiss, calculate, refresh, suppress) | 🟡 / ❌ |

> Only `alert_service.build` exists (internal). No endpoints, no read/dismiss/suppress. Effectively unimplemented as a domain.

---

## 12. Charges

| Target action | Status | Actual |
|---|---|---|
| create / get / list | ✅ | full |
| update | ❌ | no update endpoint (only status transitions) |
| soft_delete | ✅ | DELETE |
| list_client / list_business / client_summary | 🟡 | list filterable; no dedicated summary route |
| mark_open / mark_paid / mark_overdue / cancel | 🟡 | issue + mark_paid + cancel exist; mark_overdue missing |
| record_payment / cancel_payment / reconcile | ❌ | **no payments subdomain** — charge holds status directly |
| collection tasks (create/start/complete) | ❌ | none |

> **Payments model from target not implemented** — no separate payment records, no reconcile.

---

## 13. Invoices

| Target action | Status | Actual |
|---|---|---|
| create / create_for_charge | 🟡 | `attach_invoice` (attach to charge) |
| get / get_charge_invoice | ✅ | GET `/charge/{id}` |
| list / list_client | ❌ | no list endpoints |
| update | ❌ | — |
| issue / cancel / void / mark_sent | ❌ | no lifecycle |
| get_invoice_pdf | ❌ | — |

> Invoices is a thin attach-only stub vs the full accounting-doc lifecycle in target.

---

## 14. Advance Payments

| Target action | Status | Actual |
|---|---|---|
| create / get / list / list_client | ✅ | routes present |
| list_by_year / list_due | 🟡 | year filtering in repo; no dedicated route |
| update / soft_delete | ✅ | PATCH + DELETE |
| mark_open / mark_paid / mark_overdue / cancel | ❌ | no status-transition endpoints found |
| client_summary | 🟡 | KPI + overview cover most |
| generate schedule | ✅ | POST `/generate` (extra vs target) |

---

## 15. VAT

| Target action | Status | Actual |
|---|---|---|
| create / get / list / list_client / update / soft_delete | ✅ | full (work-items) |
| mark_material_missing/received | 🟡 | `materials-complete` covers received; missing not explicit |
| mark_in_progress | 🟡 | implicit on data entry |
| validate_readiness | 🟡 | service-level checks, no endpoint |
| mark_ready_for_review / send_back | ✅ | routes present |
| mark_filed | ✅ | `/file` |
| mark_paid | ❌ | no VAT mark_paid endpoint |
| reopen | ❌ | no reopen |
| summaries (client/open/missing) | 🟡 | client summary + status-summary exist |

> VAT is the most complete domain. Minor gaps: mark_paid, reopen, explicit missing-materials.

---

## 16. Annual Reports

| Target action | Status | Actual |
|---|---|---|
| create / get / list / list_client / update / soft_delete | ✅ | full |
| full lifecycle (collecting→preparation→review→pending→submitted→closed) | ✅ | transition/status/submit routes |
| send_back / reopen / cancel | 🟡 | send_back via transition; cancel via service; reopen unclear |
| income/expense lines, calculate_tax, final_balance | ✅ | financials routes + tax engine |
| summaries | ✅ | season + status summaries |

> Strong coverage. (extra vs target: annex lines, schedules, credit points, PDF export.)

---

## 17. National Insurance

| Target action | Status |
|---|---|
| **all** (records CRUD, lifecycle, summaries) | ❌ |

> No `national_insurance/` folder. Only `annual_reports/services/ni_engine.py::calculate_national_insurance` (a calculator, not a domain). Target says "target only — not yet split." Confirmed: not implemented.

---

## 18. Tax Calendar

| Target action | Status | Actual |
|---|---|---|
| create/get/update/delete events | 🟡 | entries are **generated/materialized**, not CRUD'd via API |
| generate_tax_due_dates / generate_annual_schedule | ✅ | `generate_for_year(_range)` + materialization |
| upcoming / client_calendar / overdue queries | 🟡 | groups + summary + entries routes; no per-client calendar route |

> Implemented as rules + materialization + grouped read, not as event CRUD. Different shape than target but covers generation.

---

## 19. Binders

| Target action | Status | Actual |
|---|---|---|
| create / get / list / list_client / ready_for_handover / update / soft_delete | ✅ | (create = `receive`); full coverage |
| receive_material / mark_full / reopen_capacity | ✅ | routes |
| mark_ready / revert_ready / handover / return | ✅ | routes (+ bulk variants) |
| location/capacity status / status_summary | 🟡 | tracked internally; summary partial |

> Strong coverage, plus bulk ops + history beyond target.

---

## 20. Communication

| Target action | Status | Actual |
|---|---|---|
| email (send/draft/resend/log_inbound/delivery_status) | ❌ | not implemented |
| sms / whatsapp (all) | ❌ | not implemented |
| calls (log) | 🟡 | generic correspondence entry covers logging |
| meetings (note create/update) | 🟡 | generic correspondence |
| list_client_communications / thread | ✅ | `list_correspondence_by_client` |

> Only a generic **correspondence log** exists. No channel-specific send (email/SMS/WhatsApp), no delivery status, no drafts.

---

## 21. Notifications

| Target action | Status | Actual |
|---|---|---|
| templates (create/update/delete/list) | ❌ | no template CRUD (templates are code-side renderer) |
| preview | ✅ | POST `/preview` |
| resolve_recipients | 🟡 | service `resolve_*` (internal) |
| send / send_bulk / resend_failed | 🟡 | `send` endpoint; no bulk/resend endpoints |
| policy (check/block/unblock) | 🟡 | service `can_send`; no block/unblock |
| status / list_events | 🟡 | summary + list endpoints |

---

## 22. Search

| Target action | Status | Actual |
|---|---|---|
| global_search | ✅ | GET `/` (unified `search`) |
| per-entity search (clients/legal_entities/businesses/contacts/documents/charges/tasks) | 🟡 | single global endpoint; no per-entity routes (documents has a service) |
| list_available_filters / validate_filters | ❌ | not exposed |

---

## 23. Dashboard

| Target action | Status | Actual |
|---|---|---|
| get_dashboard_summary | 🟡 | `overview` covers it |
| get_client_dashboard | ❌ | no per-client dashboard |
| get_advisor_dashboard / get_secretary_dashboard | ❌ | no role dashboards |
| (extra) tax-submissions widget | ✅ | present |

---

## 24. Work Queue

| Target action | Status | Actual |
|---|---|---|
| get_work_queue | ✅ | GET `/` |
| get_priority_actions / overdue / upcoming | 🟡 | computed inside list; no separate routes |
| filter_by_advisor / client / type | 🟡 | filters on the one endpoint, not separate |

> Functionally covered by a single filterable endpoint.

---

## 25. Actions

| Target action | Status | Actual |
|---|---|---|
| list_available_actions / get_action_definition | ❌ | no registry endpoints |
| preview_action / execute_action | ❌ | no generic execute endpoint |
| get_binder/business/charge/annual_report/vat actions | 🟡 | service functions exist (`get_*_actions`), **no api layer** |

> Action registry exists as services but is not exposed; no generic preview/execute.

---

## 26. Reports

| Target action | Status | Actual |
|---|---|---|
| aging / vat_compliance / annual_report_status / advance_payment reports | ✅ | all four endpoints |
| report status / download | 🟡 | aging export only; no generic status/download |
| dataset exports (excel/pdf/clients/documents/charges/compliance) | 🟡 | aging excel/pdf + client export (in clients); not unified |

---

## 27. Audit

| Target action | Status | Actual |
|---|---|---|
| create / get / list events | 🟡 | writer service + entity trail; no generic create/get/list endpoints |
| list_by_actor / by_entity | 🟡 | entity trail endpoint; actor via users audit-logs |
| client / entity / business audit log | 🟡 | entity trail covers; no separate business route |
| user_activity_log | ✅ | users `/audit-logs` |
| change_history | 🟡 | via entity trail |
| timeline (client/entity) | 🟡 | **timeline is its own domain** — `get_client_timeline` exists; entity timeline missing |

---

## 28. Users

| Target action | Status | Actual |
|---|---|---|
| create / get / list / update / deactivate / reactivate | ✅ | activate/deactivate routes |
| invite_user | ❌ | no invite (create only) |
| reset_user_password / update_profile | ✅ | reset-password route; profile via update |
| assign_role / remove_role | ❌ | no role assignment endpoints (role set at create/update) |
| (extra) auth: login/refresh/me/logout, forgot/reset password | ✅ | present |

---

## 29. Permissions

| Target action | Status |
|---|---|
| check_client_access / check_role_permission / list_user_permissions | ❌ |

> No `permissions/` domain. Role checks live in `core`/deps (out of scanned scope). No API.

---

## 30. Client Import

| Target action | Status | Actual |
|---|---|---|
| validate_import / preview_import | 🟡 | excel import does validation inline; no separate validate/preview endpoints |
| import_clients / commit_import | ✅ | POST `/import` (`import_clients_from_excel`) — single-step, no commit phase |

> Folded into `clients/` as Excel import. No two-phase preview→commit flow from target.

---

## Recommended build order

Driven by **dependency** (source-of-truth first) and **gap size**:

| Priority | Domain | Why |
|---|---|---|
| **P0** | **Legal Entities (2)** | Source of truth for all tax classification per target. Repo-only today. Everything VAT/NI/income-tax depends on it. Build service + API + tax-profile. |
| **P0** | **Permissions (29)** | Role enforcement is cross-cutting; target wants explicit checks. Confirm core coverage, then expose. |
| **P1** | **Contacts (4)** | Empty shell, fully specced, no dependencies. Clean greenfield. |
| **P1** | **Clients lifecycle + ownership** | freeze/close/reopen + accountant assignment — high operational value, isolated. |
| **P2** | **Documents (7)** | Required docs, folders, tagging, expiry lifecycle. Large but self-contained. |
| **P2** | **National Insurance (17)** | Split out from annual_reports; engine already exists. |
| **P2** | **Charges payments (12)** | record_payment / reconcile — needed for the Charge-as-source-of-truth model. |
| **P3** | **Communication channels (20)** | email/SMS/WhatsApp send + delivery status. Needs external providers. |
| **P3** | **Alerts (11)**, **Actions API (25)** | Expose existing service logic via endpoints. |
| **P4** | polish: Notifications templates, Tasks assignment, Reminders update/snooze, Invoices lifecycle, role dashboards | incremental gap-fills on working domains. |
