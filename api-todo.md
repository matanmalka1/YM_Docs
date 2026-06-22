# API TODO לפי אזורים
_מבוסס על gap analysis מ-OpenAPI spec | יוני 2026_

**מקרא:** 🔍 = דורש בירור/החלטה לפני תיקון · AC = Acceptance Criteria (מתי המשימה גמורה)

---

## תמונת מצב

- **פתוח לביצוע:** פריטים עם AC לא מסומן וללא ✅.
- **דורש החלטה:** פריטים עם 🔍 לפני שינוי חוזה או מימוש.
- **בוצע:** הועבר לארכיון [api-todo-done.md](archive/api-todo-done.md). נשארות בקובץ זה רק החלטות "לא למימוש"/"לא נדרש" (החלטות מוצר).

---

## פתוח / לביצוע - דומיינים

### VAT

#### 4. `POST /api/v1/vat/work-items/bulk-transition` — לא למימוש עכשיו
**החלטת מוצר:** לא מוסיפים endpoint גנרי לשינוי סטטוס מרובה. אם יעלה צורך מוצרי, לפתוח endpoint ספציפי לפעולה מוגדרת כמו bulk assign / cancel / materials-complete, עם חוקי הדומיין שלה.

#### 56. VAT CREATE רק global, לא per-client — לא נדרש עכשיו
**החלטת מוצר:** `POST /api/v1/vat/clients/{client_record_id}/work-items` הוא עקביות API בלבד ולא צורך מוצרי כרגע. לא לממש אלא אם code review עתידי מוכיח צורך אמיתי.

### Charges

#### 5. `PATCH /api/v1/charges/{charge_id}` — לא למימוש
**בעיה:** charge שנוצר לא ניתן לעריכה (סכום, תיאור, תאריך).
**AC:**
- [ ] עדכון מותר רק בסטטוס `draft` (בירור 🔍: האם לאפשר גם ב-`issued`?) — לא למימוש
- [ ] מחזיר `ChargeResponse` מעודכן + `409`  אם הסטטוס לא מאפשר עריכה  

#### 30. `POST /api/v1/charges/{charge_id}/restore`
**AC:**
- [ ] משחזר charge מבוטל/מחוק (בירור 🔍: מאיזה סטטוס ניתן לשחזר?)

### Notifications

#### 6. `PATCH /api/v1/notifications/{notification_id}` — לא למימוש
**החלטת מוצר:** Notifications הן רשומות היסטוריית תקשורת יוצאת ללקוח (`sent`/`failed`/`skipped`/`pending`) ולא inbox פנימי למשתמשי המערכת. אין מצב קריאה (`read`/`unread`) ואין צורך ב-endpoint לסימון נקרא/לא נקרא.

#### 7. `POST /api/v1/notifications/mark-all-read` — לא למימוש
**החלטת מוצר:** לא נדרש סימון גורף כנקרא, כי הדומיין לא מנהל סטטוס קריאה להתראות.

#### 8. `DELETE /api/v1/notifications/{notification_id}` — לא למימוש
**החלטת מוצר:** לא מוסיפים dismiss/delete להתראות, כי הרשומות משמשות audit trail של ניסיונות שליחה ותוכן שנשלח/נכשל. הסרה מהתצוגה רלוונטית רק אם יתווסף בעתיד Notification Center אישי, ואז צריך לפתוח החלטת מוצר נפרדת ולא למחוק את היסטוריית השליחה.

### Reminders

#### 9. `PATCH /api/v1/reminders/{reminder_id}`
**בעיה:** אין עדכון תזכורת (תאריך, טקסט).
**AC:**
- [ ] עדכון חלקי של תזכורת בסטטוס `scheduled`
- [ ] מחזיר `409` אם התזכורת כבר `fired`/`canceled`

#### 10. `DELETE /api/v1/reminders/{reminder_id}`
**בעיה:** אין מחיקה — רק cancel.
**AC:**
- [ ] בירור 🔍: האם DELETE נחוץ בנוסף ל-cancel, או ש-cancel מספיק? אם כן — `204` + soft delete

### Tax Calendar

#### 14–18. ניהול Tax Calendar (settings)
**בעיה:** settings הם read-only + bootstrap בלבד. אין יצירה/עדכון/מחיקה של חוקים ו-entries.
**AC:**
- [ ] `PATCH /settings/tax-calendar/rules/{rule_id}` — עדכון חוק
- [ ] `POST /settings/tax-calendar/rules` — יצירת חוק
- [ ] `DELETE /settings/tax-calendar/rules/{rule_id}` — מחיקה
- [ ] `PATCH /settings/tax-calendar/entries/{entry_id}` — עדכון entry
- [ ] `DELETE /settings/tax-calendar/entries/{entry_id}` — מחיקה
- [ ] בירור 🔍: עדכון חוק — האם משפיע רטרואקטיבית על entries קיימים או רק עתידיים?

### Binders

#### 21. `POST /api/v1/binders` + `PATCH /api/v1/binders/{binder_id}`
**בעיה:** binders פועלים רק דרך transitions — אין POST/PATCH ישיר.
**AC:**
- [ ] בירור 🔍: האם המודל הנוכחי (receive-only) מכוון? אם כן — לתעד ולסגור. אם לא — להוסיף POST/PATCH

#### 22. `POST /api/v1/binders/{binder_id}/restore`
**בעיה:** restore קיים ב-clients אבל לא ב-binders אחרי DELETE.
**AC:**
- [ ] משחזר binder מחוק, עקבי עם `clients/{id}/restore`

#### 58. 🔍 `BinderHandoverToClientRequest` — body אופציונלי
**בעיה:** `anyOf: [schema, null]` — POST ללא body מותר.
**AC:**
- [ ] בירור: מכוון? אם כן — לתעד. אם לא — להסיר את ה-null

#### 77. `binders/{id}/intakes/{intake_id}` — יש PATCH אבל אין DELETE
**בעיה:** אפשר לעדכן intake בודד אבל לא למחוק. אם אפשר להוסיף (receive) ולעדכן — צריך גם להסיר.
**AC:**
- [ ] `DELETE /binders/{binder_id}/intakes/{intake_id}` — מחיקת intake בודד
- [ ] מחזיר `204` + `404` אם לא קיים

### Annual Reports

#### 23. `PATCH /api/v1/annual-reports/{report_id}`
**בעיה:** אין PATCH ראשי — רק `PATCH /{id}/details`.
**AC:**
- [ ] בירור 🔍: האם `/details` מספיק או שצריך wrapper ראשי? להחליט ולתעד

#### 48. 🔍 `GET /annual-reports` vs `GET /clients/{id}/annual-reports`
**בעיה:** הראשון כבר מסנן לפי `client_record_id`, אז השני מיותר.
**AC:**
- [ ] בירור: להסיר את השני או לתעד מדוע נחוצים שניהם

#### 76. `POST /annual-reports/{id}/expenses` ו-`/income` — אין GET-list להורה
**בעיה:** לשתי הקולקציות יש רק POST — אין GET שמחזיר את שורות ההכנסה/הוצאה. השורות ניתנות ל-PATCH/DELETE, אבל אי אפשר לשלוף את הרשימה בנפרד; היא מגיעה רק מקוננת ב-`AnnualReportDetailResponse` השמן (52 שדות). חוסר עקביות בולט: `annex/{schedule}` כן יש לו GET.
**AC:**
- [ ] `GET /annual-reports/{id}/expenses` — מחזיר שורות הוצאה עם envelope אחיד
- [ ] `GET /annual-reports/{id}/income` — מחזיר שורות הכנסה
- [ ] מאפשר רענון השורות בלי טעינת הדוח המלא

### Signature Requests

#### 28. `GET /api/v1/signature-requests` — LIST כללי
**בעיה:** קיים רק `/pending`.
**AC:**
- [ ] LIST עם filter על סטטוס (כולל `pending` כמקרה פרטי)

#### 29. `POST /api/v1/signature-requests/{request_id}/cancel` — ישיר
**בעיה:** cancel קיים רק דרך `/clients/{id}/signature-requests/{id}/cancel`.
**AC:**
- [ ] cancel ישיר ללא צורך ב-client context, עקבי עם שאר ה-cancel

### Clients / Users

#### 26–27. Bulk operations ל-clients
**AC:**
- [ ] `POST /clients/bulk-patch` — עדכון מרובה, מחזיר סיכום הצלחות/כשלונות
- [ ] `POST /clients/bulk-delete` — מחיקה/ארכוב מרובה

#### 31. `DELETE /api/v1/users/{user_id}`
**בעיה:** אין DELETE — רק deactivate.
**AC:**
- [ ] בירור 🔍: האם soft delete בלבד מכוון (audit/compliance)? אם כן — לתעד ולסגור

### Reports

#### 49. `GET /reports/annual-reports` — שם מבלבל
**בעיה:** מחזיר `AnnualReportStatusReportResponse` (דו"ח סטטוס), לא רשימה.
**AC:**
- [ ] שינוי שם ל-`/reports/annual-report-status` (או דומה)


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

## פתוח / לביצוע - חוזה API רוחבי

### Cross-System Consistency

#### 72. `restore` לא עקבי
**בעיה:** קיים ב-clients/businesses, חסר ב-binders/charges/documents.
**AC:**
- [ ] restore בכל דומיין עם soft delete (מקושר לפריטים 22, 30).

---

## בוצע (ארכיון)

פריטים שהושלמו (✅) הועברו לארכיון [api-todo-done.md](archive/api-todo-done.md).
