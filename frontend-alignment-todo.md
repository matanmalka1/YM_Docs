# Frontend Alignment TODO
_Frontend gaps vs backend API | June 2026_

**Legend:** AC = Acceptance Criteria · 🔍 = requires clarification before fixing

---

## Section 1 — Missing Features (entire domain or capability absent from frontend)

### F-01 · Reminders domain — entirely missing from frontend

**Priority:** High

**Backend:** `GET /api/v1/reminders/`, `POST /api/v1/reminders/`, `GET /api/v1/reminders/{id}`, `POST /api/v1/reminders/{id}/cancel`

**Gap:** No `src/features/reminders/` directory exists. No API client, no React Query hooks, no UI wiring. The backend has a fully operational reminders domain; the frontend has zero integration.

**AC:**
- [ ] `frontend/src/features/reminders/api/endpoints.ts` defines all 4 endpoint paths (`/api/v1/reminders/`, `/api/v1/reminders/{id}`, `/api/v1/reminders/{id}/cancel`).
- [ ] `frontend/src/features/reminders/api/contracts.ts` defines:
  - `ReminderResponse { id, client_record_id, business_id?, contact_id?, action_type, remind_at, note?, status, created_by, created_at, updated_at }`
  - `CreateReminderPayload { client_record_id, business_id?, contact_id?, action_type, remind_at, note? }`
  - `ReminderListResponse` = `PaginatedResponse<ReminderResponse>`
  - `ReminderStatus` enum: `scheduled | fired | canceled`
  - `ReminderActionType` enum: snake_case values matching backend (e.g. `create_task`, `send_message`, etc.)
- [ ] `frontend/src/features/reminders/api/reminders.api.ts` exports `remindersApi` with:
  - `list(params: { page?, page_size? }): Promise<ReminderListResponse>`
  - `getById(id: number): Promise<ReminderResponse>`
  - `create(payload: CreateReminderPayload): Promise<ReminderResponse>`
  - `cancel(id: number): Promise<ReminderResponse>`
- [ ] React Query hooks exist in `frontend/src/features/reminders/hooks/`:
  - `useReminders(params)` — query key includes params; calls `remindersApi.list`
  - `useReminder(id)` — calls `remindersApi.getById`
  - `useCreateReminder()` — mutation; invalidates list on success
  - `useCancelReminder()` — mutation; invalidates list and detail on success
- [ ] `ReminderActionType` values are snake_case in both the TS enum and the API requests (backend uses snake_case per item #68 in api-todo.md).
- [ ] TypeScript compiles with no errors after adding the feature (no `any` escape hatches).

---

## Section 2 — API Contract Mismatches

### F-02 · Binders — `markReadyForHandover` and `markReadyForHandoverBulk` wrong return type

**Priority:** High

**Backend:**
- `POST /api/v1/binders/{id}/mark-ready-for-handover` → `BinderReadyForHandoverResponse { binder: BinderResponse, notification: NotificationResponse | null }`
- `POST /api/v1/binders/bulk-mark-ready-for-handover` → `{ results: BinderReadyForHandoverResponse[] }`

**Gap:** Frontend declares both as returning `BinderResponse` / `BinderResponse[]`. The `notification` field is silently dropped, so any caller that needs to display or act on the notification emitted by the backend cannot.

**File:** `frontend/src/features/binders/api/binders.api.ts` lines 57–65, `frontend/src/features/binders/api/contracts.ts`

**AC:**
- [ ] `contracts.ts` adds:
  ```ts
  export interface BinderReadyForHandoverResponse {
    binder: BinderResponse
    notification: NotificationResponse | null
  }
  export interface BinderMarkReadyForHandoverBulkResponse {
    results: BinderReadyForHandoverResponse[]
  }
  ```
- [ ] `bindersApi.markReadyForHandover(binderId)` return type changes from `Promise<BinderResponse>` to `Promise<BinderReadyForHandoverResponse>`.
- [ ] `bindersApi.markReadyForHandoverBulk(payload)` return type changes from `Promise<BinderResponse[]>` to `Promise<BinderMarkReadyForHandoverBulkResponse>`.
- [ ] Any existing callers of these two methods are updated to use `result.binder` where they previously used the result directly as `BinderResponse`.
- [ ] TypeScript compiles with no errors; no `as any` casts added.

---

### F-03 · VAT — `updateWorkItem` (PATCH) missing from API client

**Priority:** Medium

**Backend:** `PATCH /api/v1/vat/work-items/{item_id}` — accepts `VatWorkItemUpdateRequest { assigned_to?, pending_materials_note? }`, returns `VatWorkItemResponse`. Filed items return 409.

**Gap:** `vatReportsApi` in `frontend/src/features/vatReports/api/vatReports.api.ts` has no `updateWorkItem` method. Cannot update `assigned_to` or `pending_materials_note` from the frontend.

**AC:**
- [ ] `contracts.ts` adds:
  ```ts
  export interface UpdateVatWorkItemPayload {
    assigned_to?: number | null
    pending_materials_note?: string | null
  }
  ```
- [ ] `vatReportsApi.updateWorkItem(id: number, payload: UpdateVatWorkItemPayload): Promise<VatWorkItemResponse>` is added, calling `PATCH /api/v1/vat/work-items/{id}` with the payload.
- [ ] A React Query mutation hook `useUpdateVatWorkItem()` is added in the vatReports hooks (or the existing hooks file, wherever other VAT mutations live), invalidating the work item detail and list queries on success.
- [ ] TypeScript compiles with no errors.

---

### F-04 · VAT — `deleteWorkItem` (DELETE) missing from API client

**Priority:** Medium

**Backend:** `DELETE /api/v1/vat/work-items/{item_id}` — soft delete; returns 204. Only non-filed items can be deleted; filed items return 409.

**Gap:** `vatReportsApi` has no `deleteWorkItem` method. Cannot soft-delete VAT work items from the frontend.

**AC:**
- [ ] `vatReportsApi.deleteWorkItem(id: number): Promise<void>` is added, calling `DELETE /api/v1/vat/work-items/{id}`.
- [ ] A React Query mutation hook `useDeleteVatWorkItem()` is added, invalidating the list query on success.
- [ ] TypeScript compiles with no errors.

---

### F-05 · Correspondence — `CorrespondenceEntry` missing `updated_at` field

**Priority:** Medium

**Backend:** `CorrespondenceResponse` includes `updated_at: datetime | null` (added in api-todo.md item #46; migration 0005 deployed).

**Gap:** `CorrespondenceEntry` in `frontend/src/features/correspondence/api/contracts.ts` does not include `updated_at`. The field is returned by the API but the frontend type drops it silently.

**File:** `frontend/src/features/correspondence/api/contracts.ts`

**AC:**
- [ ] `CorrespondenceEntry` adds `updated_at: string | null` (ISO 8601 datetime string, nullable because it is null until first PATCH).
- [ ] Any component that displays correspondence detail can access `updated_at` without a TypeScript error.
- [ ] TypeScript compiles with no errors.

---

### F-06 · Correspondence — list endpoint missing filter params

**Priority:** Medium

**Backend:** `GET /api/v1/clients/{id}/correspondence` supports query params: `correspondence_type`, `contact_id`, `occurred_after`, `occurred_before`, `order` (in addition to `page`, `page_size`, `business_id`).

**Gap:** `correspondenceApi.list` in `frontend/src/features/correspondence/api/correspondence.api.ts` only passes `{ page, page_size, business_id }`. Five filter/sort params are never sent, making server-side filtering impossible from the frontend.

**File:** `frontend/src/features/correspondence/api/correspondence.api.ts`

**AC:**
- [ ] `contracts.ts` adds (or exports) a `ListCorrespondenceParams` interface:
  ```ts
  export interface ListCorrespondenceParams {
    page?: number
    page_size?: number
    business_id?: number | null
    correspondence_type?: CorrespondenceType | null
    contact_id?: number | null
    occurred_after?: string | null
    occurred_before?: string | null
    order?: 'asc' | 'desc'
  }
  ```
- [ ] `correspondenceApi.list(clientId, params?: ListCorrespondenceParams)` signature is updated to accept and forward all fields via `toQueryParams(params)` (same pattern as other API clients in the codebase).
- [ ] No client-side filtering that duplicates these server params is added; filtering must be server-driven.
- [ ] TypeScript compiles with no errors.

---

## Section 3 — Frontend Filter Wiring

### F-07 · Binders — wire open-binder filters to the UI

**Priority:** Medium

**Backend:** `GET /api/v1/binders/open` supports `client_record_id`, `binder_number`, `location_status`, `capacity_status`, `created_after`, and `created_before`.

**Current state:** `ListOpenBindersParams` and `bindersApi.getOpenBinders` already accept and forward all supported filters. The remaining gap is that the open-binders screen does not expose or send them.

**Files:** `frontend/src/features/binders/api/binders.api.ts`, open-binders screen, `BindersFiltersBar`

**AC:**
- [ ] The open-binders screen exposes the relevant filters and passes them to `bindersApi.getOpenBinders`.
- [ ] Filter state is stored in the URL so it survives refresh and can be shared.
- [ ] Filtering is server-driven; no in-memory filtering remains for fields sent to the endpoint.
- [ ] TypeScript compiles with no errors.

---

### F-08 · Audit — server-side filters for entity audit trails

**Priority:** Medium

**Backend:** `GET /api/v1/audit/{entity_type}/{entity_id}` supports `action`, `user_id`, `created_after`, and `created_before`.

**Gap:** `EntityAuditTrailParams` only contains pagination, and `EntityAuditTrailSection` has no action, user, or date-range filtering.

**Files:** `frontend/src/features/audit/api/contracts.ts`, `frontend/src/features/audit/api/audit.api.ts`, `frontend/src/features/audit/components/EntityAuditTrailSection.tsx`

**AC:**
- [ ] `EntityAuditTrailParams` adds `action`, `user_id`, `created_after`, and `created_before`.
- [ ] The audit trail UI adds an action selector, user picker, and date range wired to those parameters.
- [ ] Filter state is stored in the URL so it survives refresh and can be shared.
- [ ] Filtering is server-driven, not applied in memory to the current page.
- [ ] TypeScript compiles with no errors.

---

### F-09 · VAT — decide whether client work-item filters need UI

**Priority:** Low · 🔍 requires clarification

**Backend:** `GET /api/v1/vat/clients/{client_record_id}/work-items` supports `year`, `period`, `status`, `assigned_to`, `due_after`, and `due_before`.

**Current state:** `VatClientWorkItemsParams` and `vatReportsApi.listByClient` already accept and forward all supported filters. `VatClientSummaryPanel` currently uses the summary endpoint, and no client work-item table consumes `listByClient`.

**AC:**
- [ ] Determine whether the client screen needs a filterable VAT work-item table or whether the summary panel is sufficient.
- [ ] If no table is needed, document the decision and close this item without a UI change.
- [ ] If a table is needed, wire its filters to `vatReportsApi.listByClient` and store filter state in the URL.
- [ ] Filtering is server-driven; no in-memory filtering duplicates the endpoint parameters.
- [ ] TypeScript compiles with no errors if UI changes are made.

---

_Note: The following items from the original audit were already fixed before this TODO was created and do not need action:_

- **Signature requests `listPending` filters** — `ListPendingSignatureRequestsParams` in `signatureRequests/api/contracts.ts` already includes `client_record_id`, `request_type`, `signer_email`, `created_after`, `created_before`, `expires_before`. API client already uses `toQueryParams`. ✅
- **Annual reports `listReports` missing `sort_by`/`order`** — `annualReportsApi.listReports` already accepts and sends `sort_by` and `order`. ✅
- **Annual reports `listClientReports` pagination** — method already accepts `{ page?, page_size? }` and passes them. ✅
- **Notifications `getSummary` missing `business_id`** — `getSummary` already accepts `{ client_record_id?, business_id? }`. ✅
