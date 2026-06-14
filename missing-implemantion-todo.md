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
| 1 | **P0** | Documents — Binder Section | Runtime bug (404) | 🔴 Unimplemented |
| 2 | **P1** | Tasks — Standalone Page | Workflow gap | 🔴 Unimplemented |
| 3 | **P2** | Users — Edit User Form | Workflow gap | 🔴 Unimplemented |
| 4 | **P2** | VAT — Client Export Button | Dead API wrapper | 🔴 Unimplemented |
| 5 | **P2** | Binders — Year Filter | Missing param/UI | 🔴 Unimplemented |
| 6 | **P2** | Annual Reports — Sort Controls | Missing UI control | 🔴 Unimplemented |
| 7 | **P3** | Notifications — Channel Filter | Missing UI filter | 🔴 Unimplemented |
| 8 | **P3** | Notifications — Per-client Summary | Missing UI data | 🔴 Unimplemented |
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

**Priority:** P0 — Runtime bug. Live UI 404s on every binder detail drawer open.

**Type:** Runtime bug

**Backend required:** Yes
**Frontend required:** Yes (minor)
**Docs/OpenAPI required:** Yes (regenerate)
**Tests required:** Yes

---

### Problem

`BinderDocumentsSection` is rendered inside `BinderDetailDrawer`. It calls `documentsApi.listByBinder(binderId)`, which sends:

```
GET /documents/binder/{binderId}
```

This endpoint **does not exist** in `backend/openapi.json`. Every binder detail drawer request fails with 404.

---

### Current OpenAPI Endpoints

```
# Documents — existing endpoints
POST /api/v1/documents/upload
GET  /api/v1/documents/client/{client_record_id}
GET  /api/v1/documents/client/{client_record_id}/signals
GET  /api/v1/documents/client/{client_record_id}/versions
GET  /api/v1/documents/client/{client_record_id}/{document_id}
GET  /api/v1/documents/client/{client_record_id}/{document_id}/download-url
DELETE /api/v1/documents/client/{client_record_id}/{document_id}
PUT  /api/v1/documents/client/{client_record_id}/{document_id}/replace
GET  /api/v1/documents/annual-report/{report_id}

# Missing — what frontend calls but does NOT exist:
GET  /api/v1/documents/binder/{binder_id}   ← DOES NOT EXIST
```

---

### Frontend Evidence

| File | Symbol | Role |
|------|--------|------|
| `frontend/src/features/documents/api/endpoints.ts` | `documentsByBinder` | Endpoint builder pointing to non-existent URL |
| `frontend/src/features/documents/api/documents.api.ts` | `listByBinder` | API method calling `documentsByBinder` |
| `frontend/src/features/binders/components/sections/BinderDocumentsSection.tsx` | `BinderDocumentsSection` | Component calling `documentsApi.listByBinder` |
| `frontend/src/features/binders/components/drawer/BinderDetailDrawer.tsx` | `BinderDetailDrawer` | Renders `BinderDocumentsSection` unconditionally |

---

### Recommended Direction

Add the backend endpoint:

```
GET /api/v1/documents/binder/{binder_id}
```

Rationale:
- Frontend already models this correctly.
- `permanent_documents` table likely has a `binder_id` column.
- Client-side filtering of all-client documents is wasteful and unsafe.
- Response schema must match existing document list schema used by `GET /documents/client/{id}` and `GET /documents/annual-report/{id}`.

---

### Backend Work Required

- [ ] Locate the documents router. Start with `backend/` — search for `router` or `documents` in the routes/api layer.
- [ ] Locate the documents service and repository.
- [ ] Add route: `GET /api/v1/documents/binder/{binder_id}`.
- [ ] Validate binder exists — return 404 if not found.
- [ ] Enforce access control: binder must belong to a client the current user can access.
- [ ] Enforce cross-client isolation: a binder's documents must not be reachable from another client's scope.
- [ ] Query `permanent_documents` where `binder_id = {binder_id}`.
- [ ] Return the same paginated response schema used by `GET /documents/client/{id}`. Do not invent a new schema.
- [ ] Return empty list (not 404) if binder has no documents.
- [ ] Add only the repository/service layer needed. Do not add redundant wrappers.
- [ ] Follow existing router/service/repo layering rules (see `docs/architecture/`).
- [ ] Regenerate `backend/openapi.json` after adding the endpoint.

---

### Frontend Work Required

- [ ] After backend endpoint is confirmed in OpenAPI, verify `DOCUMENT_ENDPOINTS.documentsByBinder` matches the new path (`/documents/binder/{binderId}` — check for `/api/v1/` prefix stripping convention in `frontend/src/api/client.ts`).
- [ ] Keep `documentsApi.listByBinder` as-is unless path or schema changes.
- [ ] Verify `BinderDocumentsSection` handles:
  - [ ] Loading state
  - [ ] Empty list (binder has no documents)
  - [ ] Error state (non-404 errors)
  - [ ] Paginated list of documents
- [ ] If endpoint path changes from `/documents/binder/{id}` to any other path, update `DOCUMENT_ENDPOINTS.documentsByBinder` and all call sites.
- [ ] Do not add client-side filtering as a workaround. Use the backend endpoint.

---

### Docs/OpenAPI Work Required

- [ ] Confirm `backend/openapi.json` is regenerated and includes the new endpoint.
- [ ] Add the endpoint to any relevant API docs under `docs/` if an API reference exists.

---

### Tests Required

**Backend:**
- [ ] `list documents by binder — success — returns paginated docs`
- [ ] `list documents by binder — binder has no documents — returns empty list, not 404`
- [ ] `list documents by binder — binder does not exist — returns 404`
- [ ] `list documents by binder — document attached to different binder is not returned`
- [ ] `list documents by binder — cross-client access is forbidden`
- [ ] `list documents by binder — unauthenticated request returns 401`
- [ ] `list documents by binder — role access follows existing documents policy`

**Frontend:**
- [ ] `BinderDocumentsSection renders documents list from API`
- [ ] `BinderDocumentsSection shows empty state when list is empty`
- [ ] `BinderDocumentsSection shows error state on non-404 error`
- [ ] If project has contract/type tests, verify response shape is accepted

---

### Step-by-Step Implementation Plan

1. Open `backend/openapi.json` and confirm no binder documents endpoint exists.
2. In the backend, locate the documents module (router, service, repository).
3. Confirm that `permanent_documents` has a `binder_id` column by checking the Alembic migration or model definition.
4. Add the repository query: `get_by_binder_id(binder_id, page, page_size)`.
5. Add the service method if layering requires it.
6. Add the route `GET /api/v1/documents/binder/{binder_id}` with binder ownership validation.
7. Write backend tests.
8. Regenerate `backend/openapi.json`.
9. Confirm `frontend/src/features/documents/api/endpoints.ts` `documentsByBinder` path matches the new OpenAPI path.
10. Run frontend typecheck.
11. Open a binder detail drawer in the app — confirm no 404, documents render or empty state appears.
12. Run all tests.

---

### Acceptance Criteria

- [ ] `GET /api/v1/documents/binder/{binder_id}` exists in `backend/openapi.json`.
- [ ] Opening binder detail drawer sends a request to the correct endpoint (not a 404 path).
- [ ] Binder with documents shows document list.
- [ ] Binder with no documents shows clean empty state.
- [ ] Cross-client access to binder documents is forbidden.
- [ ] Backend tests pass.
- [ ] Frontend typecheck passes.
- [ ] No existing document endpoints are broken.

---

### Risk Notes / Edge Cases

- Confirm `permanent_documents.binder_id` is nullable or always set for binder-attached docs. If the column does not exist, the data model needs updating first.
- If binder access control follows a different pattern from client access control, ensure the new endpoint is consistent.
- Do not skip IDOR check: route must verify current user can access the binder's owning client.

---

### Final Verification Commands

```bash
# Confirm endpoint exists in OpenAPI after backend change
jq '.paths | keys[] | select(contains("binder"))' backend/openapi.json

# Confirm frontend endpoint builder matches
grep -n "documentsByBinder\|binder" frontend/src/features/documents/api/endpoints.ts

# Confirm listByBinder is called correctly
grep -rn "listByBinder" frontend/src/features/binders/

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## P1 — Tasks: No Standalone Tasks List UI

**Priority:** P1 — Workflow gap. Backend fully supports task listing with rich filters. `tasksApi.list` is dead code.

**Type:** Workflow gap

**Backend required:** No (verify schema is complete)
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** Yes (frontend)

---

### Problem

`tasksApi.list` is defined in `frontend/src/features/tasks/api/tasks.api.ts` with full filter param support but is **never called** from any component, hook, or page. There is no `/tasks` route. Task interactions exist only through the Work Queue surface, which does not expose the full task list independently.

---

### Current OpenAPI Endpoints

```
GET  /api/v1/tasks
  Query params: status, priority, assigned_to_user_id, assigned_role,
                source_domain, source_id, due_before, due_after, page, page_size

GET  /api/v1/tasks/{task_id}
POST /api/v1/tasks
PATCH /api/v1/tasks/{task_id}
DELETE /api/v1/tasks/{task_id}
POST /api/v1/tasks/{task_id}/complete
POST /api/v1/tasks/{task_id}/cancel
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/tasks/api/tasks.api.ts` | `tasksApi.list` | Defined, never called |
| `frontend/src/features/tasks/api/tasks.api.ts` | `tasksApi.get` | Called only in `useWorkQueueActions` for edit |
| `frontend/src/features/tasks/components/TaskModal.tsx` | `TaskModal` | Exists; used via work queue |
| `frontend/src/router/AppRoutes.tsx` | `/tasks` route | Does not exist |
| `frontend/src/features/tasks/` | `TasksPage` | Does not exist |

---

### Backend Work Required

- [ ] None expected.
- [ ] Verify `GET /api/v1/tasks` returns correct paginated response shape — check against `TaskListResponse` in `frontend/src/features/tasks/api/contracts.ts`.
- [ ] If a filter is missing from the backend, add it. Otherwise do not change backend.

---

### Frontend Work Required

- [ ] Add `TasksPage` component at `frontend/src/features/tasks/pages/TasksPage.tsx`.
- [ ] Add route `/tasks` in `frontend/src/router/AppRoutes.tsx`.
- [ ] Call `tasksApi.list` with params driven by URL state or component state (follow existing list page convention — check `ClientsPage`, `VatWorkItems`, or `ChargesPage` for the established pattern).
- [ ] Expose filter controls for:
  - [ ] `status` (enum — source from existing `TaskStatus` type)
  - [ ] `priority` (enum — source from existing type)
  - [ ] `assigned_to_user_id` (user picker — reuse existing user list hook)
  - [ ] `assigned_role`
  - [ ] `source_domain`
  - [ ] `due_before` / `due_after` (date range)
- [ ] Support pagination (follow `PaginatedDataTable` or existing pattern).
- [ ] Reuse `TaskModal` for create and edit.
- [ ] Reuse existing complete/cancel/delete task action logic (from `useWorkQueueActions` or extract shared action hooks).
- [ ] Add nav/sidebar link. Follow existing `Navbar.constants.ts` pattern.
- [ ] Persist filters in URL params for shareable links (follow existing convention).
- [ ] Reset page to 1 on filter change.
- [ ] Export `TasksPage` from `frontend/src/features/tasks/index.ts`.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

**Frontend:**
- [ ] `TasksPage renders task list from API`
- [ ] `TasksPage filter by status`
- [ ] `TasksPage filter by priority`
- [ ] `TasksPage pagination`
- [ ] `TasksPage opens TaskModal on row click`
- [ ] `TasksPage create task opens TaskModal in create mode`

---

### Step-by-Step Implementation Plan

1. Read an existing list page (e.g., `frontend/src/features/vatReports/pages/VatWorkItems.tsx`) to confirm the pattern for URL-driven filters + paginated table.
2. Read `frontend/src/features/tasks/api/contracts.ts` to understand `TaskListParams`, `TaskListResponse`, `Task` shape.
3. Create `frontend/src/features/tasks/pages/TasksPage.tsx`.
4. Create a `useTasksPage` hook in `frontend/src/features/tasks/hooks/useTasksPage.ts` that calls `tasksApi.list`.
5. Add filter controls following the `FilterPanel` convention.
6. Add table using `PaginatedDataTable` or the existing table component.
7. Add row actions: open/edit, complete, cancel, delete.
8. Add route to `AppRoutes.tsx`.
9. Add nav link per `Navbar.constants.ts` convention.
10. Export from `index.ts`.
11. Run frontend typecheck.
12. Verify `/tasks` loads and filters work.

---

### Acceptance Criteria

- [ ] `/tasks` route exists and renders a paginated tasks list.
- [ ] User can filter by: status, priority, assigned user, assigned role, source domain, due date range.
- [ ] User can open and edit a task via `TaskModal`.
- [ ] User can complete, cancel, and delete tasks from the list.
- [ ] `tasksApi.list` is called from `TasksPage` and is no longer dead code.
- [ ] Page/task state resets correctly on filter change.
- [ ] Frontend typecheck passes.

---

### Risk Notes / Edge Cases

- Do not duplicate task action logic from `useWorkQueueActions`. Extract shared mutation hooks if needed.
- If `source_domain` filter is multi-value in backend, ensure the params serializer handles it correctly (check `toSearchParams` in `tasks.api.ts`).

---

### Final Verification Commands

```bash
# Confirm tasksApi.list is called
grep -rn "tasksApi.list" frontend/src/

# Confirm route exists
grep -n "tasks" frontend/src/router/AppRoutes.tsx

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## P2 — Users: PATCH Update Has No UI Form

**Priority:** P2 — Workflow gap. Admins cannot edit users after creation.

**Type:** Workflow gap

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** Yes (frontend)

---

### Problem

`PATCH /api/v1/users/{user_id}` exists in OpenAPI and `usersApi.update` exists in the frontend API client, but no edit form or modal is wired anywhere in `UsersPage`. Admins can create, activate/deactivate, and reset passwords but cannot change user fields (name, role, etc.) after creation.

---

### Current OpenAPI Endpoint

```
PATCH /api/v1/users/{user_id}
operationId: update_user_api_v1_users__user_id__patch
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/users/api/users.api.ts` | `usersApi.update` | Defined, never called |
| `frontend/src/features/users/pages/UsersPage.tsx` | User row actions | Has: create, activate/deactivate, reset-password. No edit. |
| `frontend/src/features/users/` | `UserEditModal` | Does not exist |

---

### Backend Work Required

- [ ] None expected.
- [ ] Confirm the PATCH request body schema from OpenAPI (`UpdateUserPayload` in `frontend/src/features/users/api/contracts.ts`). Verify which fields are editable.

---

### Frontend Work Required

- [ ] Identify which fields are included in `UpdateUserPayload` (check `frontend/src/features/users/api/contracts.ts`).
- [ ] Determine if user list response (`UserResponse`) includes all fields needed to pre-fill the edit form, or if `usersApi.getById` is needed.
  - If list row has all fields: pre-fill from row data, do not call `getById`.
  - If list row is missing fields: call `usersApi.getById` to pre-fill.
- [ ] Add an "Edit" row action to the users table in `UsersPage`.
- [ ] Create `UserEditModal` at `frontend/src/features/users/components/UserEditModal.tsx` (or reuse `CreateUserModal` only if it is cleanly separable — prefer a separate modal).
- [ ] Modal must:
  - [ ] Pre-fill all editable fields from existing user data.
  - [ ] Call `usersApi.update(userId, payload)` on submit.
  - [ ] Invalidate users list query on success.
  - [ ] Show loading state during submit.
  - [ ] Show error on failure.
- [ ] Preserve activate/deactivate and reset-password as separate explicit actions — do not merge them into the edit form.
- [ ] Export new component from `frontend/src/features/users/index.ts`.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

**Frontend:**
- [ ] `UserEditModal pre-fills fields from existing user data`
- [ ] `UserEditModal submits PATCH with updated fields`
- [ ] `UserEditModal shows error on API failure`
- [ ] `UsersPage edit row action opens UserEditModal`

---

### Step-by-Step Implementation Plan

1. Read `frontend/src/features/users/api/contracts.ts` — identify `UpdateUserPayload` fields.
2. Read `frontend/src/features/users/components/CreateUserModal.tsx` for component pattern.
3. Read `frontend/src/features/users/pages/UsersPage.tsx` for row action pattern.
4. Create `UserEditModal` with form fields matching `UpdateUserPayload`.
5. Add "Edit" action to user row.
6. Wire `usersApi.update` in modal submit handler.
7. On success, invalidate `usersQK.list`.
8. Export from `index.ts`.
9. Run frontend typecheck.
10. Verify edit flow works end-to-end.

---

### Acceptance Criteria

- [ ] Admin can open an edit form for any user from `UsersPage`.
- [ ] Edit form pre-fills current user data.
- [ ] Submitting calls `usersApi.update`.
- [ ] User list refreshes after successful update.
- [ ] Activate/deactivate remains a separate action — not part of the edit form.
- [ ] `usersApi.update` is no longer dead code.
- [ ] Frontend typecheck passes.

---

### Risk Notes / Edge Cases

- Role changes (if editable) may have security implications. Confirm the backend validates role assignments against the current user's own role.
- If `usersApi.getById` is needed for pre-fill and its response differs from the list item shape, keep both usages clearly separated.

---

### Final Verification Commands

```bash
# Confirm usersApi.update is called
grep -rn "usersApi.update" frontend/src/

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## P2 — VAT: Client Export Button Missing

**Priority:** P2 — Dead API wrapper with clear product value.

**Type:** Dead API wrapper / missing UI

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** Yes (frontend)

---

### Problem

`GET /api/v1/vat/clients/{client_record_id}/export` exists in OpenAPI and `vatReportsApi.exportClientVat` is implemented in the frontend API client, but no UI component calls it. Users cannot download a VAT export per client.

---

### Current OpenAPI Endpoint

```
GET /api/v1/vat/clients/{client_record_id}/export
operationId: export_vat_client_api_v1_vat_clients__client_record_id__export_get
Query params: format (excel | pdf), year
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/vatReports/api/vatReports.api.ts` | `vatReportsApi.exportClientVat` | Defined, never called |
| `frontend/src/features/vatReports/` | Export button | Does not exist in any VAT component |

---

### Backend Work Required

- [ ] None expected.

---

### Frontend Work Required

- [ ] Locate the client VAT tab or section. Start with `frontend/src/features/vatReports/` and `frontend/src/features/clients/components/details/`.
- [ ] Add an export button (PDF and/or Excel — follow `vatReportsApi.exportClientVat` signature: `format: 'excel' | 'pdf'`, `year: number`).
- [ ] The button must:
  - [ ] Show loading state while exporting.
  - [ ] Be disabled during export.
  - [ ] Trigger blob download on success (pattern already in `vatReportsApi.exportClientVat`).
  - [ ] Show error toast on failure.
- [ ] Identify which `year` to pass. Options: currently selected tax year from context, or a year picker adjacent to the button. Follow existing convention (check advance payments or annual reports export for year source).

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

**Frontend:**
- [ ] `Export button appears in client VAT UI`
- [ ] `Clicking export triggers vatReportsApi.exportClientVat`
- [ ] `Button is disabled during export`
- [ ] `Error toast shown on failure`

---

### Step-by-Step Implementation Plan

1. Search `frontend/src/features/vatReports/` and `frontend/src/features/clients/` for the client VAT tab component.
2. Read `vatReportsApi.exportClientVat` signature in `vatReports.api.ts`.
3. Determine where the export button belongs (VAT tab header or toolbar).
4. Determine year source (URL param, context, or picker).
5. Add export button.
6. Wire `vatReportsApi.exportClientVat(clientId, format, year)`.
7. Add loading/disabled state.
8. Add error handler.
9. Run frontend typecheck.
10. Test download end-to-end.

---

### Acceptance Criteria

- [ ] Client VAT tab (or appropriate section) has a visible export button.
- [ ] Clicking the button triggers `vatReportsApi.exportClientVat` and downloads the file.
- [ ] Button shows loading state during download.
- [ ] Button is disabled while export is in progress.
- [ ] Error is surfaced if export fails.
- [ ] `vatReportsApi.exportClientVat` is no longer dead code.
- [ ] Frontend typecheck passes.

---

### Risk Notes / Edge Cases

- If no year is available in context, a sensible default (current tax year) or a picker must be provided. Do not silently pass `undefined` if the param is required.

---

### Final Verification Commands

```bash
# Confirm exportClientVat is called
grep -rn "exportClientVat" frontend/src/

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

---

## P2 — Binders: Year Filter Missing

**Priority:** P2 — Missing param/UI. OpenAPI supports `year` but frontend omits it.

**Type:** Missing param/UI

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** No (minor change)

---

### Problem

`GET /api/v1/binders` accepts a `year` query parameter, but `ListBindersParams` in `frontend/src/features/binders/types.ts` does not include `year`, and the binders page filter panel has no year filter.

---

### Current OpenAPI Endpoint

```
GET /api/v1/binders
Query params: location_status, capacity_status, client_record_id, query, client_name,
              binder_number, year, page, page_size, sort_by, order
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/binders/types.ts` | `ListBindersParams` | Missing `year` |
| Binders page filter panel | Year filter | Does not exist |

---

### Backend Work Required

- [ ] None expected.

---

### Frontend Work Required

- [ ] Add `year?: number` to `ListBindersParams` in `frontend/src/features/binders/types.ts`.
- [ ] Add year filter to the binders page filter panel.
  - Locate binders filter component (search `frontend/src/features/binders/` for filter/toolbar).
  - Add a numeric year input or select — follow existing convention (check `AnnualReportsPage` for year filter pattern).
- [ ] Persist `year` in URL params if binders page uses URL state for filters.
- [ ] Pass `year` in `bindersApi.list(params)` call.
- [ ] Include year in filter reset behavior.
- [ ] Validate that `year` is a valid number before sending.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

- [ ] None strictly required for a minor param addition, but if existing filter tests exist, add year case.

---

### Step-by-Step Implementation Plan

1. Read `frontend/src/features/binders/types.ts` — confirm `ListBindersParams` shape.
2. Add `year?: number` to `ListBindersParams`.
3. Locate binders page filter component.
4. Add year filter input.
5. Wire `year` into URL params (follow existing binders filter URL convention).
6. Pass `year` in `bindersApi.list`.
7. Add reset support.
8. Run frontend typecheck.
9. Verify year filter appears and sends `year` in request.

---

### Acceptance Criteria

- [ ] `ListBindersParams` includes `year?: number`.
- [ ] Binders page filter panel has a year filter.
- [ ] Selecting a year re-fetches binders with `year` query param.
- [ ] Clearing year removes param from request.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
# Confirm year is in type
grep -n "year" frontend/src/features/binders/types.ts

# Confirm year sent in request
grep -rn "year" frontend/src/features/binders/

# Frontend typecheck
cd frontend && npx tsc --noEmit
```

---

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

## P3 — Notifications: Channel Filter Missing

**Priority:** P3 — Missing UI filter for an available API param.

**Type:** Missing UI filter

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** No

---

### Problem

`GET /api/v1/notifications` accepts a `channel` filter param, but `NotificationsPage` does not expose a channel filter to the user. All other supported filters (trigger, status, created_after, created_before, triggered_by, client_record_id) are wired.

---

### Current OpenAPI Endpoint

```
GET /api/v1/notifications
Supported params: client_record_id, business_id, status, trigger, channel, triggered_by,
                  created_after, created_before, page, page_size
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/notifications/pages/NotificationsPage.tsx` | Filter panel | Wires all params except `channel` |
| `frontend/src/features/notifications/api/contracts.ts` | `ListNotificationsParams` | Likely missing `channel` field |

---

### Backend Work Required

- [ ] None expected.

---

### Frontend Work Required

- [ ] Check `ListNotificationsParams` in `frontend/src/features/notifications/api/contracts.ts` — add `channel?: string` if missing.
- [ ] Source channel options: check the OpenAPI schema for `channel` enum or existing `NOTIFICATION_CHANNEL_OPTIONS` constant. If no constant exists, derive from schema or API response values.
- [ ] Add `channel` dropdown filter to `NotificationsPage` filter panel.
- [ ] Wire `channel` to URL param: read from `searchParams`, set via `setFilter`.
- [ ] Pass `channel` to `notificationsApi.listPaginated` params.
- [ ] Include `channel` in filter reset.

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

- [ ] None strictly required for a minor filter addition.

---

### Step-by-Step Implementation Plan

1. Read `frontend/src/features/notifications/api/contracts.ts` — check `ListNotificationsParams`.
2. Add `channel?: string` if missing.
3. Determine channel values from OpenAPI schema or existing constants.
4. Add `channel` dropdown to `NotificationsPage` filter panel (follow pattern of existing `trigger` or `status` dropdown).
5. Wire to URL param and API call.
6. Run frontend typecheck.

---

### Acceptance Criteria

- [ ] `NotificationsPage` filter panel includes a `channel` dropdown.
- [ ] Selecting a channel sends `channel` param in API request.
- [ ] Clearing channel removes param.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
grep -n "channel" frontend/src/features/notifications/pages/NotificationsPage.tsx
grep -n "channel" frontend/src/features/notifications/api/contracts.ts
cd frontend && npx tsc --noEmit
```

---

---

## P3 — Notifications: Per-Client Summary Not Shown in Client Tab

**Priority:** P3 — Missing UI data. Per-client notification status summary not shown in client notifications tab.

**Type:** Missing UI data

**Backend required:** No
**Frontend required:** Yes
**Docs/OpenAPI required:** No
**Tests required:** No

---

### Problem

`GET /api/v1/notifications/summary` accepts `client_record_id` and `business_id`, returning per-scope status counts. The global `NotificationBell` calls it with no params. The client `NotificationsTab` lists per-client notifications but does not call `getSummary({ client_record_id })` to show status count badges.

---

### Current OpenAPI Endpoint

```
GET /api/v1/notifications/summary
Query params: client_record_id, business_id
```

---

### Frontend Evidence

| File | Symbol | Status |
|------|--------|--------|
| `frontend/src/features/notifications/api/notifications.api.ts` | `notificationsApi.getSummary` | Exists |
| `frontend/src/components/layout/NotificationBell.tsx` | Global bell | Calls `getSummary()` with no params |
| `frontend/src/features/notifications/components/NotificationsTab.tsx` | `NotificationsTab` | Lists client notifications; no summary call |

---

### Backend Work Required

- [ ] None expected.

---

### Frontend Work Required

- [ ] In `NotificationsTab` (or its backing hook), add a query calling `notificationsApi.getSummary({ client_record_id })`.
- [ ] Display compact summary counts (e.g., "sent: N / failed: N" or status badges).
- [ ] Add loading and empty handling.
- [ ] Do not change the global `NotificationBell` behavior — it uses a separate call with no params.
- [ ] Source the display pattern from existing summary/KPI card conventions (check `AdvancePaymentsKPICards` or `ClientStatusCard` for style reference).

---

### Docs/OpenAPI Work Required

- [ ] None expected.

---

### Tests Required

- [ ] None strictly required.

---

### Step-by-Step Implementation Plan

1. Read `frontend/src/features/notifications/components/NotificationsTab.tsx`.
2. Identify where the client `id` is available.
3. Add `useQuery` for `notificationsApi.getSummary({ client_record_id: clientId })`.
4. Add compact summary display above or beside the notifications list.
5. Run frontend typecheck.

---

### Acceptance Criteria

- [ ] Client notifications tab calls `getSummary({ client_record_id })`.
- [ ] Summary counts are displayed (at minimum: sent, failed).
- [ ] Global notification bell behavior is unchanged.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
grep -n "getSummary" frontend/src/features/notifications/components/NotificationsTab.tsx
cd frontend && npx tsc --noEmit
```

---

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

## P3 — Annual Reports: GET /details — Verify Before Acting

**Priority:** P3 — Requires verification. Do not implement until the investigation below is completed.

**Type:** Verify first

**Backend required:** Unknown
**Frontend required:** Unknown
**Docs/OpenAPI required:** Unknown
**Tests required:** Unknown

---

### Problem

OpenAPI exposes:
```
GET  /api/v1/annual-reports/{report_id}/details
PATCH /api/v1/annual-reports/{report_id}/details
```

Frontend has:
- `annualReportsApi.patchReportDetails` (PATCH) — wired in UI
- No `getReportDetails` (GET) — never called

The main report detail page appears to rely on `GET /api/v1/annual-reports/{report_id}`. If the main GET response embeds the same fields as the details sub-resource, no frontend change is needed. If it returns different or additional fields, silent data loss may occur.

---

### Required Investigation

- [ ] Compare the response schema of `GET /api/v1/annual-reports/{report_id}` vs `GET /api/v1/annual-reports/{report_id}/details` in `backend/openapi.json`.
  ```bash
  jq '.paths["/api/v1/annual-reports/{report_id}"].get.responses["200"].content["application/json"].schema' backend/openapi.json
  jq '.paths["/api/v1/annual-reports/{report_id}/details"].get.responses["200"].content["application/json"].schema' backend/openapi.json
  ```
- [ ] If the schemas are identical or details is a subset of the main report: **no frontend change needed. Document decision here.**
- [ ] If the details schema contains fields not in the main report: add `annualReportsApi.getReportDetails(reportId)` and call it from the report detail page in the appropriate section.

---

### Decision Checklist

After investigation, record the outcome:

- [ ] **Decision A:** GET /details response is a subset of GET /{id}. No action needed. Mark as resolved.
- [ ] **Decision B:** GET /details returns additional fields not in GET /{id}. Implement `getReportDetails` and wire in report detail page.

---

### Acceptance Criteria (conditional on Decision B)

- [ ] If Decision B: `annualReportsApi.getReportDetails` exists and is called from the annual report detail page.
- [ ] No fields available via GET /details are silently missing from the UI.
- [ ] Frontend typecheck passes.

---

### Final Verification Commands

```bash
# Compare schemas
jq '.paths["/api/v1/annual-reports/{report_id}"].get.responses["200"]' backend/openapi.json
jq '.paths["/api/v1/annual-reports/{report_id}/details"].get.responses["200"]' backend/openapi.json
```

---

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
