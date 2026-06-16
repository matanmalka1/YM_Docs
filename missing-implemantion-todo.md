# Missing Implementation TODO

> Source: OpenAPI vs Frontend UI Implementation Audit (2026-06-14)
> Scope: YM Tax CRM — internal tool, no external consumers, no backward-compat constraints.

---

## Executive Summary

An audit of `backend/openapi.json` against `frontend/src` identified 12 actionable findings ranging from a live runtime 404 to missing UI controls for fully implemented backend features.

**One finding is a runtime bug that breaks existing UI today (P0).**
The remaining findings are workflow gaps, missing filters, or dead API wrappers.

Reminders are explicitly excluded from this document.

---

## Priority Order

| # | Priority | Area | Type | Status |
|---|----------|------|------|--------|
| 1 | **P0** | Documents — Binder Section | Runtime bug (404) | ✅ Resolved (2026-06-16) — endpoint implemented, see `docs/domains/permanent-documents.md` |
| 2 | **P1** | Tasks — Standalone Page | Workflow gap | ✅ Resolved (2026-06-16) — already implemented |
| 3 | **P2** | Users — Edit User Form | Workflow gap | ✅ Resolved (2026-06-16) — already implemented |
| 4 | **P2** | VAT — Client Export Button | Dead API wrapper | ✅ Resolved (2026-06-16) — already implemented |
| 5 | **P2** | Binders — Year Filter | Missing param/UI | ✅ Resolved (2026-06-16) — already implemented |
| 6 | **P2** | Annual Reports — Sort Controls | Missing UI control | 🔴 Unimplemented |
| 7 | **P3** | Notifications — Channel Filter | Missing UI filter | ⏸️ Skipped (2026-06-16) — WhatsApp not yet integrated |
| 8 | **P3** | Notifications — Per-client Summary | Missing UI data | ✅ Resolved (2026-06-16) |
| 9 | **P3** | Correspondence — `getById` Missing | API parity | 🔴 Unimplemented |
| 10 | **P3** | Annual Reports — GET /details | Verify first | ⚠️ Verify before acting |
| 11 | **P3** | Signature Requests — Standalone Page | Verify first | ⚠️ Verify before acting |
| 12 | **Low** | Dead API wrappers cleanup | Cleanup | 🔴 Unimplemented |

---

## Do Not Implement Blindly

The following findings require investigation **before** any implementation:

- **Finding 10** — `GET /annual-reports/{report_id}/details`: Verify whether the response schema differs from `GET /annual-reports/{report_id}`. If it is a subset, no frontend work is needed.
- **Finding 11** — Signature Requests standalone page: Verify whether a list-all backend endpoint exists before building any UI. Do not build a frontend page against a non-existent endpoint.

---

## Out of Scope / Excluded

### Reminders

All `/api/v1/reminders/*` endpoints are excluded from this TODO.

```
GET  /api/v1/reminders/
POST /api/v1/reminders/
GET  /api/v1/reminders/{reminder_id}
POST /api/v1/reminders/{reminder_id}/cancel
```

UI/UX for reminders is not in scope for this audit cycle. Do not add reminder tasks to this file.

---

---

## P0 — Documents: BinderDocumentsSection Calls Non-Existent Endpoint

> **✅ RESOLVED (2026-06-16).** `GET /api/v1/documents/binder/{binder_id}` is implemented: router (`backend/app/documents/permanent_documents/api/permanent_documents.py:108`), service `list_binder_documents` (raises `BINDER.NOT_FOUND` 404), repo `list_by_binder_page`. Confirmed in `backend/openapi.json`. Frontend `documentsByBinder`/`listByBinder` already call the correct path; `npx tsc --noEmit` clean. Canonical detail: `docs/domains/permanent-documents.md`.

---

> **✅ RESOLVED (2026-06-16).** Tasks list UI already implemented: `/tasks` route (`AppRoutes.tsx:159`), `TasksPage` (`features/tasks/pages/TasksPage.tsx`), `useTasksPage` → `useTasks` → `tasksApi.list`. `tasksApi.list` is no longer dead code.

---

## P2 — Users: PATCH Update Has No UI Form

> **✅ RESOLVED (2026-06-16).** Already implemented: `EditUserModal` (`features/users/components/EditUserModal.tsx`), wired via `useUsersPage` → `usersApi.update` (`useUsersPage.ts:56`), edit row action in `UsersPage.tsx`, exported from `features/users/index.ts`. `usersApi.update` is no longer dead code. `tsc --noEmit` passes.


---

## P2 — VAT: Client Export Button Missing

> **✅ RESOLVED (2026-06-16).** Already implemented: `VatExportButtons` component wired into `VatClientActionBar` and `VatWorkItemSummaryBar`, backed by `useVatExport` hook calling `vatReportsApi.exportClientVat`. Loading/disabled state and error toast present. `vatReportsApi.exportClientVat` is no longer dead code. Tests not added (skipped per decision).

---

## P2 — Binders: Year Filter Missing

> **✅ RESOLVED (2026-06-16).** Already implemented: `ListBindersParams` includes `year?: number` (`frontend/src/features/binders/types.ts`), `BindersFiltersBar` has a year filter wired through `useBindersFilters` → URL params → `bindersApi.list`. No UI work remains.

---

## P2 — Annual Reports: Sort Controls Missing

**Priority:** P2 — Missing UI control. Sort params typed in API client but never set by user.

**Type:** Missing UI control

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** No (minor UI addition)

---

### Problem

`GET /api/v1/annual-reports` accepts `sort_by` and `order` params. `annualReportsApi.listReports` includes these in its params type. But no UI control in `AnnualReportsPage` allows the user to set sort order.

---

### Current OpenAPI Endpoint

```
GET /api/v1/annual-reports
Query params: tax_year, client_record_id, status, page, page_size,
              sort_by (tax_year | status | filing_deadline | created_at | client_record_id),
              order (asc | desc)
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/annualReports/api/annualReports.api.ts` | `sort_by`, `order` in `listReports` params type | Typed but never set from user action |
| `frontend/src/features/annualReports/pages/AnnualReportsPage.tsx` | Sort controls | Do not exist |

---

### Backend Work Required

- [ ] None expected.

---

### Frontend Work Required

- [ ] Read `frontend/src/features/annualReports/pages/AnnualReportsPage.tsx` to understand current table structure.
- [ ] Add sortable column headers for:
  - [ ] `tax_year`
  - [ ] `status`
  - [ ] `filing_deadline`
  - [ ] `created_at`
  - [ ] Client column (maps to `client_record_id` sort if a client name column exists)
- [ ] Toggle: clicking an already-sorted column toggles `order` between `asc` and `desc`.
- [ ] Persist `sort_by` and `order` in URL params if page uses URL state.
- [ ] Reset page to 1 on sort change.
- [ ] Show visual sort indicator (up/down arrow or equivalent).
- [ ] Follow existing sortable column pattern — check other list pages (e.g., `ClientsPage` or `ChargesPage`) for the established convention.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

- [ ] None strictly required. If existing page tests exist, add sort toggle case.

---

### Step-by-Step Implementation Plan

1. Read `AnnualReportsPage.tsx` and its table definition.
2. Identify which columns are sortable per the API.
3. Read existing sortable column implementation in another list page (e.g., `ClientsPage`).
4. Add `sort_by` and `order` to URL params / page state.
5. Add sort indicators to column headers.
6. Wire sort params into `annualReportsApi.listReports` call.
7. Reset page on sort change.
8. Run frontend typecheck.
9. Verify sort triggers refetch with correct params.

---

### Acceptance Criteria

- [ ] `AnnualReportsPage` has sortable column headers for all supported `sort_by` values.
- [ ] Clicking a column header sorts the list.
- [ ] Clicking the same header again reverses the order.
- [ ] Active sort is visually indicated.
- [ ] Sort state persists in URL params.
- [ ] Page resets to 1 on sort change.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
# Confirm sort_by/order are wired
grep -n "sort_by\|order" frontend/src/features/annualReports/pages/AnnualReportsPage.tsx

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## P3 — Notifications: Channel Filter Missing — ⏸️ SKIPPED

**Priority:** P3 — Missing UI filter for an available API param.

**Status:** Skipped 2026-06-16 by decision. Only channels are `whatsapp`/`email` (`NotificationChannel` enum); WhatsApp delivery is not yet integrated (no provider wired beyond the disabled `WhatsAppChannel` stub), so a channel filter has no real-world value yet. Revisit once WhatsApp delivery ships.

---

## P3 — Notifications: Per-Client Summary Not Shown in Client Tab — ✅ DONE

**Priority:** P3 — Missing UI data. Per-client notification status summary not shown in client notifications tab.

**Status:** Implemented 2026-06-16. Added `useNotificationsSummary` hook (`frontend/src/features/notifications/hooks/useNotificationsSummary.ts`) calling `notificationsApi.getSummary({ client_record_id })`. `NotificationsTab` now renders sent/pending/failed badges when `summary.total > 0`. Global `NotificationBell` untouched. `tsc --noEmit` passes.

---

## P3 — Correspondence: `getById` Missing from API Client

**Priority:** P3 — API parity. Backend endpoint exists; frontend API client lacks the method.

**Type:** API parity

**Backend required:** No
**Frontend required:** Yes (API client only, no UI required)
**Docs/OpenAPI required:** No
**Tests required:** No

---

### Problem

`GET /api/v1/clients/{client_record_id}/correspondence/{correspondence_id}` exists in OpenAPI but `correspondenceApi` in `frontend/src/features/correspondence/api/correspondence.api.ts` has no `getById` method. The API client has list/create/update/delete only.

This does not currently break any UI (no component needs single-record fetch), but it is an incomplete API client that blocks future detail/deep-link functionality.

---

### Current OpenAPI Endpoint

```
GET /api/v1/clients/{client_record_id}/correspondence/{correspondence_id}
operationId: get_correspondence_api_v1_clients__client_record_id__correspondence__correspondence_id__get
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/correspondence/api/correspondence.api.ts` | `correspondenceApi` | Has list, create, update, delete. Missing `getById`. |
| `frontend/src/features/correspondence/api/endpoints.ts` | `correspondenceById` | Endpoint builder exists and is already used by update/delete |

---

### Backend Work Required

- [ ] None.

---

### Frontend Work Required

- [ ] Add `getById` to `correspondenceApi` in `frontend/src/features/correspondence/api/correspondence.api.ts`:
  ```ts
  getById: async (clientId: number, id: number): Promise<CorrespondenceEntry> => {
    const response = await api.get<CorrespondenceEntry>(
      CORRESPONDENCE_ENDPOINTS.correspondenceById(clientId, id)
    )
    return response.data
  }
  ```
- [ ] Use existing `CORRESPONDENCE_ENDPOINTS.correspondenceById` which already handles the path.
- [ ] Do not add UI unless a natural detail/deep-link scenario exists. This is API client parity only.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

- [ ] None strictly required for adding a method that isn't called.

---

### Step-by-Step Implementation Plan

1. Read `frontend/src/features/correspondence/api/correspondence.api.ts`.
2. Read `frontend/src/features/correspondence/api/endpoints.ts`.
3. Add `getById` method.
4. Run frontend typecheck.

---

### Acceptance Criteria

- [ ] `correspondenceApi.getById(clientId, id)` exists and compiles.
- [ ] It uses `CORRESPONDENCE_ENDPOINTS.correspondenceById`.
- [ ] Existing correspondence list/create/update/delete are unchanged.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
grep -n "getById" frontend/src/features/correspondence/api/correspondence.api.ts
cd frontend && npx tsc --noEmit
```

---

---

## P3 — Annual Reports: GET /details — RESOLVED (Decision A, no action needed)

**Resolved 2026-06-16:** Compared `AnnualReportDetailResponse` (main GET) vs `ReportDetailResponse` (details GET) in `backend/openapi.json`. Every field in `ReportDetailResponse` (`pension_contribution`, `donation_amount`, `other_credits`, `client_approved_at`, `internal_notes`, `amendment_reason`, `created_at`, `updated_at`) already exists in `AnnualReportDetailResponse`. Details response is a strict subset — main report GET fully covers it. No silent data loss. No frontend change needed.


---

## P3 — Signature Requests: Standalone System-Wide Page — Verify Before Acting

**Priority:** P3 — Requires product decision and backend verification before any UI is built.

**Type:** Verify first

**Backend required:** Possibly
**Frontend required:** Possibly
**Docs/OpenAPI required:** Possibly
**Tests required:** Possibly

---

### Problem

There is no system-wide `/signature-requests` page. The dashboard exposes only pending requests via `GET /api/v1/signature-requests/pending`. A completed/declined/all-status list view may require a backend endpoint that does not yet exist.

---

### Current OpenAPI Endpoints

```
GET  /api/v1/signature-requests/pending
  operationId: list_pending_requests_api_v1_signature_requests_pending_get

GET  /api/v1/signature-requests/{request_id}
  operationId: get_signature_request_api_v1_signature_requests__request_id__get

GET  /api/v1/clients/{client_record_id}/signature-requests
  operationId: list_client_signature_requests_...
```

**No list-all endpoint with status filter exists in OpenAPI.**

---

### Required Investigation

- [ ] Determine product intent: Is a system-wide signature requests page needed?
  - **Option A:** Pending-only page — use existing `GET /signature-requests/pending`. Build frontend only.
  - **Option B:** All-statuses page — requires new backend endpoint `GET /api/v1/signature-requests?status=&page=&page_size=`.
- [ ] If Option B: backend must add the endpoint first. OpenAPI must be regenerated. Then build frontend.

---

### Do Not Build

- [ ] Do not build a frontend page that calls a non-existent backend endpoint.
- [ ] Do not build an all-status list using client-side state or workarounds.

---

### Implementation Plan (Option A — Pending-only page)

If product decision is Option A:

**Backend:** No work needed.

**Frontend:**
- [ ] Add `/signature-requests` route.
- [ ] Build `SignatureRequestsPage` using `signatureRequestsApi.listPending`.
- [ ] Display pending requests with cancel action.
- [ ] Add nav/sidebar link.
- [ ] Export from `frontend/src/features/signatureRequests/index.ts`.

---

### Implementation Plan (Option B — All-status page)

If product decision is Option B:

**Backend:**
- [ ] Add `GET /api/v1/signature-requests` with params: `status`, `client_record_id`, `page`, `page_size`.
- [ ] Enforce access control.
- [ ] Regenerate `backend/openapi.json`.

**Frontend:**
- [ ] Add API client method for list-all.
- [ ] Add `/signature-requests` route and `SignatureRequestsPage`.
- [ ] Add status filter.
- [ ] Add pagination.
- [ ] Add nav link.

---

### Acceptance Criteria (conditional on product decision)

- [ ] A clear product decision is documented here: Option A or Option B.
- [ ] If Option A: `/signature-requests` page shows pending requests using existing endpoint.
- [ ] If Option B: backend endpoint exists in OpenAPI before any frontend work starts.
- [ ] No UI is built against a non-existent endpoint.

---

### Final Verification Commands

```bash
# Confirm what exists
jq '.paths | keys[] | select(contains("signature"))' backend/openapi.json
```

---

---

## Dead or Inconsistent Frontend API Code — Cleanup

### Overview

The following API symbols are dead code: either unused or calling a non-existent endpoint. Clean these up as part of related feature work, not as standalone tasks.

---

### Item 1: `documentsByBinder` / `listByBinder` / `BinderDocumentsSection`

**Files:**
- `frontend/src/features/documents/api/endpoints.ts` — `documentsByBinder`
- `frontend/src/features/documents/api/documents.api.ts` — `listByBinder`
- `frontend/src/features/binders/components/sections/BinderDocumentsSection.tsx`
- `frontend/src/features/binders/components/drawer/BinderDetailDrawer.tsx`

**Problem:** `documentsByBinder` points to a non-existent endpoint. This causes runtime 404.

**Action:**
- [ ] If P0 backend endpoint is added: update `documentsByBinder` path if needed; keep `listByBinder` and `BinderDocumentsSection`.
- [ ] If P0 backend endpoint is NOT added: remove `documentsByBinder`, `listByBinder`, and `BinderDocumentsSection`; remove from `BinderDetailDrawer`; remove from `documentsQK` if a query key exists.
- [ ] Do not leave a reference to a non-existent route in production code.

---

### Item 2: `documentsApi.getDocument`

**File:** `frontend/src/features/documents/api/documents.api.ts`

**Problem:** `getDocument` (GET single document by client + document ID) is defined but never called from any component or hook.

**Action:**
- [ ] If a document detail/metadata refresh scenario is being built: wire it there.
- [ ] If no such scenario exists: remove `getDocument` and its `documentDetail` endpoint entry in `DOCUMENT_ENDPOINTS` (verify `documentDetail` is not also used by `updateDocument` or `deleteDocument` before removing — check all usages).
- [ ] Run typecheck after removal.

---

### Item 3: `usersApi.getById`

**File:** `frontend/src/features/users/api/users.api.ts`

**Problem:** `getById` is defined but never called.

**Action:**
- [ ] When P2 (User Edit Form) is implemented: use `getById` to pre-fill edit modal if the list row lacks all required fields. This would wire the method and resolve the dead code.
- [ ] If the edit form can pre-fill from the list row without a separate GET: remove `getById` after confirming no usage.
- [ ] Do not pre-emptively remove before the edit form is built.

---

### Item 4: `tasksApi.list`

**File:** `frontend/src/features/tasks/api/tasks.api.ts`

**Problem:** `tasksApi.list` is defined but never called.

**Action:**
- [ ] Wire in P1 (TasksPage). Do not remove.
- [ ] After P1 is complete: confirm `tasksApi.list` is called and the dead-code status is cleared.

---

### Cleanup Acceptance Criteria

- [ ] No frontend code calls a URL path that does not exist in `backend/openapi.json`.
- [ ] Dead wrappers that are about to be wired (items 3, 4) are not removed prematurely.
- [ ] If item 2 is removed, `DOCUMENT_ENDPOINTS.documentDetail` removal does not break `updateDocument` or any other caller — verify with grep before removing.
- [ ] Frontend typecheck passes after any removal.

---

### Final Verification Commands

```bash
# Check all document endpoint usages before any removal
grep -rn "documentDetail\|getDocument" frontend/src/

# Check usersApi.getById usage before P2 edit form
grep -rn "usersApi.getById\|getById" frontend/src/features/users/

# Check tasksApi.list usage after P1
grep -rn "tasksApi.list" frontend/src/

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## Final Checklist

### Backend

- [ ] P0: Add `GET /api/v1/documents/binder/{binder_id}` endpoint with access control
- [ ] P0: Validate binder exists and cross-client isolation enforced
- [ ] P0: Write backend tests for new endpoint
- [ ] P3 (Sig. Requests, if Option B): Add `GET /api/v1/signature-requests` list endpoint

### Frontend

- [ ] P0: Verify `documentsByBinder` path matches new backend endpoint (or remove if not added)
- [ ] P0: Verify `BinderDocumentsSection` handles loading/empty/error states correctly
- [ ] P1: Create `TasksPage` at `frontend/src/features/tasks/pages/TasksPage.tsx`
- [ ] P1: Add `/tasks` route to `frontend/src/router/AppRoutes.tsx`
- [ ] P1: Add filter controls for all supported task params
- [ ] P1: Add nav/sidebar link for tasks
- [ ] P1: Wire `tasksApi.list` (removes dead-code status)
- [ ] P2: Create `UserEditModal` at `frontend/src/features/users/components/UserEditModal.tsx`
- [ ] P2: Add Edit row action in `UsersPage`
- [ ] P2: Wire `usersApi.update` (removes dead-code status)
- [ ] P2: Add VAT client export button in client VAT tab
- [ ] P2: Wire `vatReportsApi.exportClientVat` (removes dead-code status)
- [ ] P2: Add `year` to `ListBindersParams` in `frontend/src/features/binders/types.ts`
- [ ] P2: Add year filter UI to binders page
- [ ] P2: Add sortable column headers to `AnnualReportsPage`
- [ ] P2: Wire `sort_by`/`order` to URL params in `AnnualReportsPage`
- [ ] P3: Add `channel` filter to `NotificationsPage`
- [ ] P3: Add per-client `getSummary` call in `NotificationsTab`
- [ ] P3: Add `correspondenceApi.getById`
- [ ] P3: Verify `GET /annual-reports/{id}/details` vs `GET /annual-reports/{id}` schema (Decision A or B)
- [ ] P3: Implement details fetch if Decision B
- [ ] P3: Implement signature requests page after product decision (Option A or B)
- [ ] Low: Remove or wire `documentsApi.getDocument` after P0 decision
- [ ] Low: Wire or remove `usersApi.getById` after P2 edit form decision
- [ ] Low: Confirm `tasksApi.list` is no longer dead after P1

### Docs / OpenAPI

- [ ] P0: Regenerate `backend/openapi.json` after adding binder documents endpoint
- [ ] P3 (Sig. Requests, if Option B): Regenerate `backend/openapi.json` after adding list-all endpoint
- [ ] P3 (Annual Reports details): Document schema comparison decision in this file under Finding 10

### Tests

- [ ] P0 backend: list by binder — success, empty, 404, cross-client isolation, auth
- [ ] P0 frontend: `BinderDocumentsSection` renders, empty state, error state
- [ ] P1 frontend: `TasksPage` renders, filters, pagination, task modal
- [ ] P2 frontend: `UserEditModal` pre-fill, submit, error
- [ ] P2 frontend: VAT export button trigger, loading, error
