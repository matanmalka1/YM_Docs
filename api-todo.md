# API TODO לפי אזורים
_מבוסס על gap analysis מ-OpenAPI spec | יוני 2026_

**מקרא:** 🔍 = דורש בירור/החלטה לפני תיקון · AC = Acceptance Criteria (מתי המשימה גמורה)

---

## תמונת מצב

- **פתוח לביצוע:** פריטים עם AC לא מסומן וללא ✅.
- **דורש החלטה:** פריטים עם 🔍 לפני שינוי חוזה או מימוש.
- **בוצע:** פריטים עם ✅ ו-AC מסומן, כולל פריטי חוזה רוחביים שהועברו מהחלק הפתוח.

---

## פתוח / לביצוע - דומיינים

### VAT

#### 2. `PATCH /api/v1/vat/work-items/{item_id}`
**בעיה:** אין דרך לעדכן work item אחרי יצירה (סטטוס, הערות, שדות פנימיים).
**AC:**
- [ ] endpoint מקבל `VatWorkItemUpdateRequest` עם שדות אופציונליים
- [ ] עדכון חלקי (שדה בודד) לא מאפס שדות אחרים
- [ ] מחזיר `VatWorkItemResponse` מעודכן + `404` אם לא קיים

#### 3. `DELETE /api/v1/vat/work-items/{item_id}`
**בעיה:** אין מחיקת work item.
**AC:**
- [ ] מחיקה רכה (soft delete) עקבית עם שאר הדומיינים
- [ ] מחזיר `204` בהצלחה, `404` אם לא קיים
- [ ] פריט מחוק לא מופיע ב-LIST כברירת מחדל

#### 4. `POST /api/v1/vat/work-items/bulk-transition`
**בעיה:** אין שינוי סטטוס מרובה — צריך לעבור פריט-פריט.
**AC:**
- [ ] מקבל רשימת IDs + transition יעד
- [ ] מחזיר סיכום הצלחות/כשלונות לכל פריט (כמו `BulkChargeActionResponse`)
- [ ] transition לא חוקי לפריט בודד לא מפיל את כל ה-batch

#### 56. 🔍 VAT CREATE רק global, לא per-client
**בעיה:** בניגוד לדומיינים אחרים שיש להם `POST /clients/{id}/X`.
**AC:**
- [ ] בירור: להוסיף `POST /vat/clients/{id}/work-items` לעקביות?

### Charges

#### 5. `PATCH /api/v1/charges/{charge_id}`
**בעיה:** charge שנוצר לא ניתן לעריכה (סכום, תיאור, תאריך).
**AC:**
- [ ] עדכון מותר רק בסטטוס `draft` (בירור 🔍: האם לאפשר גם ב-`issued`?)
- [ ] מחזיר `ChargeResponse` מעודכן + `409` אם הסטטוס לא מאפשר עריכה

#### 30. `POST /api/v1/charges/{charge_id}/restore`
**AC:**
- [ ] משחזר charge מבוטל/מחוק (בירור 🔍: מאיזה סטטוס ניתן לשחזר?)

### Notifications

#### 6. `PATCH /api/v1/notifications/{notification_id}` — mark read/unread
**בעיה:** אין endpoint לסימון notification כנקרא. ה-UI כנראה עוקף את זה בצד הלקוח.
**AC:**
- [ ] מאפשר עדכון סטטוס קריאה
- [ ] מחזיר ה-notification מעודכן + `404` אם לא קיים

#### 7. `POST /api/v1/notifications/mark-all-read`
**בעיה:** אין סימון גורף.
**AC:**
- [ ] מסמן את כל ה-notifications של המשתמש כנקראו
- [ ] מחזיר מספר הפריטים שעודכנו

#### 8. `DELETE /api/v1/notifications/{notification_id}` — dismiss
**בעיה:** אין הסרה/dismiss.
**AC:**
- [ ] מסיר notification מהתצוגה
- [ ] מחזיר `204` + `404` אם לא קיים

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

### Documents

#### 11. `GET /api/v1/documents/client/{client_record_id}/{document_id}`
**בעיה:** אפשר לרשום, להעלות, למחוק ולהחליף מסמך — אבל אין שליפת מסמך בודד.
**AC:**
- [ ] מחזיר metadata של מסמך יחיד + `404`

#### 12. `PATCH /api/v1/documents/client/{client_record_id}/{document_id}`
**בעיה:** אין עדכון metadata (שם, סוג, תגיות) בלי להחליף את הקובץ.
**AC:**
- [ ] עדכון metadata בלבד, ללא העלאה מחדש
- [ ] לא משנה את גרסת הקובץ

#### 13. `GET /api/v1/documents/binder/{binder_id}`
**בעיה:** יש `documents/client/{id}` ו-`documents/annual-report/{id}` אבל לא לפי binder.
**AC:**
- [ ] מחזיר מסמכים המשויכים ל-binder, באותו envelope כמו שאר רשימות המסמכים

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

### Tasks

#### 19. `GET /api/v1/clients/{client_record_id}/tasks`
**בעיה:** אין שליפת משימות לפי לקוח — חובה לסנן ידנית.
**AC:**
- [ ] מחזיר משימות מסוננות ל-client_record_id, עם pagination אחיד

#### 32–33. Bulk operations ל-tasks
**AC:**
- [ ] `POST /tasks/bulk-complete` — השלמה מרובה
- [ ] `POST /tasks/bulk-assign` — שיוך מחדש מרובה

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

### Frontend Wiring (המשך ל-#52)

ה-backend של #52 הושלם: הפילטרים החדשים קיימים בשרת וב-OpenAPI, ו-`frontend/src/types/generated.ts` חודש (typecheck ירוק). אף מסך עדיין לא **שולח** את הפרמטרים החדשים — כל הקריאות נשארו `page`/`page_size` בלבד. השינוי additive ו-optional, ולכן אין שבירה. הפריטים הבאים מחווטים את ה-UI לפי צורך תפעולי אמיתי.

#### 78. חיווט binders/open ל-`getOpenBinders`
**בעיה:** `BindersFiltersBar` (query/binder_number/location_status/year) קיים ומחובר לרשימת ה-binders הראשית בלבד; מסך ה-open binders שולף בלי פילטרים למרות שהשרת תומך בהם.
**מיקום:** `frontend/src/features/binders/api/binders.api.ts:85` (`getOpenBinders`), צרכן `BindersFiltersBar`.
**AC:**
- [ ] `getOpenBinders` מקבל ושולח `client_record_id`, `binder_number`, `location_status`, `capacity_status`, `created_after`, `created_before`.
- [ ] סרגל הפילטרים של open binders מחווט לפרמטרים האלה ושומר state ב-URL (refresh/share).
- [ ] אין סינון client-side in-memory שנשאר על מה שעבר לשרת.

#### 79. סינון server-side ל-audit trail
**בעיה:** `EntityAuditTrailSection` מציג את כל רשומות ה-audit; אין סינון לפי action/user, וה-fetch לא ממונף את `action`/`user_id`/`created_after`/`created_before` החדשים.
**מיקום:** `frontend/src/features/audit/api/audit.api.ts:7` (`getEntityAuditTrail`), טייפ `EntityAuditTrailParams`, צרכן `EntityAuditTrailSection`.
**AC:**
- [ ] `EntityAuditTrailParams` מורחב ל-`action`, `user_id`, `created_after`, `created_before`.
- [ ] נוסף UI סינון (action dropdown / user picker / טווח תאריכים) המחווט לפרמטרים, עם state ב-URL.
- [ ] הסינון רץ בשרת (לא client-side על דף בודד).

#### 80. ניצול filters ב-vat client work-items
**בעיה:** `listByClient` שולף את כל work-items של הלקוח בלי פרמטרים; השרת תומך עכשיו ב-`year`/`period`/`status`/`assigned_to`/`due_after`/`due_before`. כרגע `VatClientSummaryPanel` משתמש ב-summary endpoint, וה-list endpoint תת-מנוצל.
**מיקום:** `frontend/src/features/vatReports/api/vatReports.api.ts:121` (`listByClient`).
**AC:**
- [ ] 🔍 בירור: האם מסך הלקוח צריך טבלה מסוננת (year/period/status/assigned_to/due range) או שה-summary panel מספיק? אם אין צורך — לתעד ולסגור בלי שינוי UI.
- [ ] אם נדרש: `listByClient` מקבל ושולח את הפילטרים, עם UI ו-state ב-URL.

---

## בוצע כבר - חוזה API רוחבי

### Response Envelopes / Errors

#### 42. Response envelopes לא עקביים ✅ בוצע
**בעיה:** 4 וריאנטים שונים של רשימה.
**החלטה:** הסטנדרט הוא `PaginatedResponse[T]` = `{items, page, page_size, total}` לפי [api-contracts.md](architecture/api-contracts.md). `total_pages` **מוסר** (נגזר מ-`total/page_size`, מחושב בצד לקוח); aliases אסורים (ADR 0001 / entry-point.md).
**AC:**
- [x] `VatWorkItemListResponse` — `items/total/page/page_size` (guard: `backend/tests/vat/test_vat_list_envelope_schema.py`)
- [x] `VatInvoiceListResponse` — `items/total/page/page_size` (collection child bounded; envelope לעקביות; guard באותו קובץ)
- [x] `BinderIntakeListResponse` — בוטל; `GET /binders/{id}/intakes` מחזיר `PaginatedResponse[BinderIntakeResponse]` עם `items`
- [x] `CorrespondenceListResponse` — `total_pages` הוסר; envelope תקני (guard: `backend/tests/communications/test_correspondence_schemas.py`)

**בוצע:** הנירמול עצמו נחת בקומיט `a0c0ee9a fix(api): make list response envelopes consistent (#42)` (backend + `openapi.json` + frontend `generated.ts` + הצרכן `BinderIntakesSection.tsx` קורא `data.items`). בסבב הסגירה נוסף guard schema-level חסר ל-VAT (binder/correspondence כבר היו מכוסים).

#### 53. אין schema גנרי לשגיאות עסקיות ✅ בוצע
**בעיה:** רק `HTTPValidationError` (422) מוגדר. אין תיעוד מלא ל-400/403/409/500.
**AC:**
- [x] schema אחיד `ErrorEnvelope { error: { code, message, details, request_id? } }`
- [x] משויך ל-responses השגיאה בכל ה-endpoints
- [x] ה-frontend מסתמך על מבנה ידוע דרך `getApiErrorBody()`

**בוצע (sweep מלא):** המטריצה הקנונית של סטטוס-שגיאה לכל endpoint תועדה ב-[architecture/error-doc-matrix.md](architecture/error-doc-matrix.md) (נגזרה מ-raise sites אמיתיים, מאומתת מול `exception_handlers.py`). מנגנון כפול:
- **`401`/`403` מוזרקים גלובלית** ב-`build_openapi` ([app/core/openapi.py](../backend/app/core/openapi.py)): `401` לכל op שאינו public (לפי `public_endpoints.py`); `403` לכל op עם dependency של `require_role(...)` (זוהה דרך הליכה על `route.dependant`). 188/201 ה-ops הם role-gated. שני ה-ops הלא-public היחידים ללא `require_role` (`GET /auth/me`, `POST /auth/logout`) לא מגיעים ל-`ForbiddenError` בשירות.
- **`400`/`404`/`409`/`500` מתועדים per-route** דרך ה-helpers מ-`core/openapi_responses.py`. `404` כבר נסחף (פריט 25). `500` מתועד רק ב-`DOCUMENT.UPLOAD_FAILED` (upload/replace); export-ים של excel/PDF נשארים ללא 500 מתועד (env `ImportError`, לא חוזה עסקי).
- **בדיקות רגרסיה:** `test_openapi_auth_error_docs.py` (הזרקת 401/403 + exclusion ל-public + רפרנס ל-`ErrorEnvelope`), `test_error_doc_coverage_matrix.py` (כל op מתעד בדיוק את ה-400/409/500 מהמטריצה — אין חסר ואין עודף), בנוסף ל-`test_error_openapi_schema.py` הקיים.
- `openapi.json` + frontend `generated.ts` יוצאו מחדש; backend scoped suite + frontend typecheck/lint/format/test ירוקים.

### Filters / Search / Pagination

#### 50. חיפוש טקסטואלי — 4 שמות שונים ✅ בוצע
**בעיה:** `search`, `query`, `client_name`, `client_search` — לאותה פעולה.
**AC:**
- [x] שם אחיד אחד (`search`) בכל ה-endpoints הרלוונטיים לחיפוש הגלובלי
- [x] ה-frontend מעודכן

**בוצע:** `GET /api/v1/search` נורמל ל-`search` כפרמטר הטקסטואלי הרחב היחיד, ונוספה תמיכה ב-`client_id` לסינון מדויק לפי לקוח. `query` ו-`client_name` הוסרו מחוזה ה-search ולא מופיעים יותר ב-OpenAPI עבור endpoint זה. חיפוש לפי שם לקוח לא הוסר: הקלדה בתיבת החיפוש הראשית עדיין נשלחת כ-`search` וממשיכה להתאים רשומות לקוח לפי שם/זהות דרך מימוש החיפוש הקיים. סינון מתקדם לפי לקוח עבר ל-client picker ששולח `client_id`, וה-autocomplete עצמו מחפש לקוחות דרך `GET /clients?search=...`.

**בדיקות/חוזה:** נוספו בדיקות ל-`/search?search=<client name>`, ל-`search + client_id` במסמכים, למניעת זליגת מסמכים בין לקוחות, ול-OpenAPI שמוודא ש-`search`/`client_id` קיימים ו-`query`/`client_name`/`client_search` אינם קיימים ב-`/api/v1/search`. `openapi.json` ו-`frontend/src/types/generated.ts` חודשו.

#### 51. תאריכי טווח — 7 פטרנים ✅ בוצע
**בעיה:** `from`/`to`, `from_date`/`to_date`, `date_from`/`date_to`, `issued_after`/`issued_before`, `due_after`/`due_before`, `start_year`/`end_year`, `from_year`/`to_year`.
**AC:**
- [x] פטרן אחיד אחד (`{field}_after`/`{field}_before`) בכל ה-endpoints
- [x] ה-frontend מעודכן

**בוצע:** טווחי תאריכים/שנים נורמלו ל-`{field}_after`/`{field}_before`: user audit logs עברו ל-`created_after`/`created_before`; tax-calendar bootstrap/summary עברו ל-`tax_year_after`/`tax_year_before`; ותיעוד הדומיינים עודכן עבור communications/notifications/users/tax-calendar.

#### 52. 16 endpoints לרשימות ללא filters ✅ בוצע
**בעיה:** fetch עיוור ברשימות תפעוליות שעלולות לגדול.
**AC:**
- [x] בירור לכל אחד מאלה שעלולים לגדול: `GET /binders/open`, `/signature-requests/pending`, `/clients/{id}/annual-reports`, `/vat/clients/{id}/work-items`, `/clients/{id}/businesses`, `/audit/{entity_type}/{entity_id}` — האם צריך filters?

**בוצע:** אחרי בירור, נוספו filters רק לרשימות שבהן יש צורך תפעולי בסינון בתוך queue/ציר אירועים גדל:
- `GET /api/v1/binders/open` — `client_record_id`, `binder_number`, `location_status`, `capacity_status`, `created_after`, `created_before`; הרשימה נשארת open-only ולא מחזירה `handed_over`.
- `GET /api/v1/signature-requests/pending` — `client_record_id`, `request_type`, `signer_email`, `created_after`, `created_before`, `expires_before`.
- `GET /api/v1/vat/clients/{client_record_id}/work-items` — `year`, `period`, `status`, `assigned_to`, `due_after`, `due_before`.
- `GET /api/v1/audit/{entity_type}/{entity_id}` — `action`, `user_id`, `created_after`, `created_before`.

הוחלט שאין צורך ב-filters נוספים עבור `GET /api/v1/clients/{client_record_id}/annual-reports` ו-`GET /api/v1/clients/{client_record_id}/businesses`: אלו רשימות child שכבר scoped ללקוח יחיד, קטנות יחסית, והצורך שנמצא עבורן הוא יציבות/דטרמיניזם ולא סינון. לכן עודכן רק מיון יציב: annual reports לפי `tax_year desc, id desc`; businesses לפי `opened_at asc, id asc`.

#### 74. `page_size` max לא עקבי — 100 מול 200 ✅ בוצע
**בעיה:** 25 endpoints מגבילים ל-100, 16 ל-200, ו-`GET /notifications` ללא max כלל. אין הגיון בחלוקה — `GET /annual-reports`=200 אבל `GET /clients/{id}/annual-reports`=100, לאותו סוג ישות.
**AC:**
- [x] מקסימום אחיד אחד לכל ה-endpoints: `200`
- [x] `GET /notifications` מקבל max מוגדר
- [x] בדיקה: בקשה מעל המקסימום מוחזרת עם `422`

**בוצע:** נוסף `app/core/pagination.py` עם `MAX_PAGE_SIZE = 200` ו-`DEFAULT_PAGE_SIZE = 50`, וכל פרמטרי `page_size` ב-routes הציבוריים עברו ל-`le=MAX_PAGE_SIZE` תוך שמירה על ברירות המחדל הקיימות. `GET /api/v1/notifications` עבר מ-enum של `25 | 50` לוולידציה מספרית `1..200` עם default `25`. חוזה ה-API עודכן כך שברירת המחדל הסטנדרטית נשארת לפי endpoint, והמקסימום האחיד הוא `200`.

**בדיקות/חוזה:** נוספה בדיקת חוזה ממוקדת ל-pagination ב-`tests/core/api/test_pagination_contract.py`, ועודכנו בדיקות קיימות שהצפינו את הגבולות הישנים. `openapi.json` חודש ונסרק: כל 44 פרמטרי `page_size` מתועדים עם `minimum: 1` ו-`maximum: 200`; notifications מתועד עם `default: 25`.

#### 69. `NotificationPageSize` — enum `[25, 50]` ✅ בוצע
**בעיה:** pagination הוגבל ל-2 ערכים דרך enum.
**AC:**
- [x] ההגבלה הוסרה במסגרת פריט 74: `GET /api/v1/notifications` משתמש ב-`page_size` מספרי עם `minimum: 1`, `maximum: 200`, `default: 25`.

### Type Conflicts

#### 61. `occurred_at` — אותו שדה, שני טיפוסים ✅ בוצע
**בעיה:** `CorrespondenceCreateRequest`=`date-time`, `CorrespondenceUpdateRequest`=`string` גולמי.
**AC:**
- [x] `CorrespondenceUpdateRequest.occurred_at` משתמש ב-`ApiDateTime` / `datetime`
- [x] OpenAPI מייצר `format: date-time`
- [x] בדיקות schema/API קיימות מאמתות עדכון חלקי ושימור `occurred_at`

**בוצע:** אומת בקוד:
- `backend/app/communications/schemas/correspondence.py` — `CorrespondenceUpdateRequest.occurred_at: ApiDateTime | None = None`, עם validator שמקבל `datetime | None`.
- `backend/app/communications/services/correspondence_service.py` ו-`backend/app/communications/repositories/correspondence_repository.py` — `occurred_at` מטופל כ-`datetime` ב-create/update/list filters.
- `backend/tests/communications/test_correspondence_schemas.py` — בדיקות ל-`CorrespondenceUpdateRequest` עבור `occurred_at` (null אסור, omission הוא partial, future date נדחה).
- `backend/tests/communications/api/test_correspondence_update_delete.py` — PATCH חלקי לא מאפס `occurred_at`.
- `backend/openapi.json` — `CorrespondenceUpdateRequest.properties.occurred_at.anyOf[0].format = "date-time"`.

#### 63. שדות פיננסיים — `string` גולמי ב-Update ✅ בוצע
**בעיה:** `gross_amount`, `paid_amount`, `expected_amount`, `recognition_rate`, `other_credits` הם `decimal` ב-Create/Response אבל `string` גולמי ב-Update.
**AC:**
- [x] `VatInvoiceUpdateRequest.gross_amount` משתמש ב-`ApiDecimal`
- [x] `AdvancePaymentUpdateRequest.paid_amount` ו-`expected_amount` משתמשים ב-`ApiDecimal`
- [x] `ExpenseLineUpdateRequest.recognition_rate` משתמש ב-`ApiDecimal`
- [x] `AnnualReportDetailUpdateRequest.other_credits` משתמש ב-`ApiDecimal`
- [x] OpenAPI מייצר לכל השדות `type: string`, `format: decimal`

**בוצע:** אומת בקוד:
- `backend/app/vat/schemas/vat_invoice_update.py` — `gross_amount: ApiDecimal | None`.
- `backend/app/advance_payments/schemas/advance_payment.py` — `AdvancePaymentUpdateRequest.paid_amount` ו-`expected_amount` הם `ApiDecimal | None`.
- `backend/app/annual_reports/schemas/annual_report_financials.py` — `ExpenseLineUpdateRequest.recognition_rate: ApiDecimal | None`.
- `backend/app/annual_reports/schemas/annual_report_detail.py` — `AnnualReportDetailUpdateRequest.other_credits: ApiDecimal | None`.
- `backend/tests/core/test_api_types.py` — guard ל-OpenAPI decimal serialization/contract.
- `backend/tests/core/test_update_request_conventions.py` — כל ה-Update schemas הרלוונטיים כלולים ב-guard של `NonEmptyUpdateMixin`; `recognition_rate` נבדק גם כ-non-nullable.
- `backend/openapi.json` — ארבעת ה-Update schemas הנ"ל מציגים `format: decimal` עבור השדות.

---

## פתוח / לביצוע - חוזה API רוחבי

### Type Conflicts

#### 59. 🔍 `created_at` — `date-time` מול `date`
**בעיה:** כמעט כולם `date-time`, אבל `VatInvoiceResponse` הוא `date`.
**AC:**
- [ ] בירור: חשבונית VAT צריכה שעה? להחליט ולאחד.

#### 60. 🔍 `filing_deadline` — 3 טיפוסים
**בעיה:** `string` ללא format / `date-time` / `date` — לאותו שדה לוגי.
**AC:**
- [ ] בירור: מהו הטיפוס הנכון? לאחד בכל 4 ה-schemas.

#### 62. 🔍 `amount` — `decimal` מול `string` גולמי
**בעיה:** רוב המקומות `decimal`, `AttentionBoardItem` גולמי.
**AC:**
- [ ] בירור: האם זה סכום כספי? אם כן — `decimal`.

#### 64. 🔍 `id` — `integer` מול `string`
**בעיה:** רוב הישויות `integer`. `WorkQueueItem.id` הוא `string` — **מאושר כמכוון** (יש לו regex `^\w+:\d+$`, composite ID כמו `vat_work_item:42`). נשאר רק `AttentionBoardItem.id` כ-`string` ללא הסבר.
**AC:**
- [ ] בירור: האם `AttentionBoardItem.id` גם composite? אם כן — להוסיף regex ולתעד. אם לא — `integer`.

#### 65. 🔍 `counterparty_id` — `string` במקום `integer`
**בעיה:** ב-VatInvoice schemas.
**AC:**
- [ ] בירור: מספר עוסק/ח.פ (string מכוון) או FK פנימי?

#### 66. 🔍 שיעורים כ-`number` במקום `decimal`
**בעיה:** `collection_rate`, `completion_rate`, `compliance_rate`, `effective_rate`, `credit_points`, `rate` כ-float.
**AC:**
- [ ] בירור: float מקובל לשיעורים, או `decimal` כמו שאר הכספים (סיכון עיגול)?

### Schema / Enum Design

#### 57. inline enum ב-correspondence `order`
**בעיה:** `['asc','desc']` inline במקום `$ref`.
**AC:**
- [ ] enum מוגדר כ-schema נפרד ומופנה אליו

#### 67. 🔍 `ExpenseCategory` vs `ExpenseCategoryType`
**בעיה:** שני enums לסיווג הוצאות, 7 ערכים משותפים + ערכים שונים.
**AC:**
- [ ] בירור: מה ההבדל ואיפה כל אחד? לאחד או לתעד.

#### 68. `ReminderActionType` — UPPER_CASE חריג
**בעיה:** היחיד ב-UPPER_CASE מתוך 52 enums (השאר snake_case).
**AC:**
- [ ] המרה ל-`snake_case` (`create_task` וכו') + עדכון frontend.

### Cross-System Consistency

#### 70. `client_id` vs `client_record_id`
**בעיה:** שני path params מעורבבים ב-spec.
**AC:**
- [x] איחוד ל-`client_record_id` (ה-anchor התפעולי) בכל ה-paths + frontend.

#### 72. `restore` לא עקבי
**בעיה:** קיים ב-clients/businesses, חסר ב-binders/charges/documents.
**AC:**
- [ ] restore בכל דומיין עם soft delete (מקושר לפריטים 22, 30).

### Bugs

#### 73. שדות `required` שהם גם `nullable` — 18 שדות (סתירה לוגית)
**בעיה:** שדה לא יכול להיות גם חובה וגם null. זה מבלבל codegen — ב-TypeScript ייווצר `field: string | null` אך מסומן חובה. כנראה נובע מ-Pydantic models עם `Optional` ללא default.
**מושפעים:** `VatPeriodRow` (8 שדות: `filed_at`, `period_type`, `final_vat_amount`, `is_overdue`, `submission_deadline`, `statutory_deadline`, `extended_deadline`, `days_until_deadline`), `EntityAuditLogResponse` + `VatAuditLogResponse` (`old_value`, `new_value`, `note`), `TaxCalculationSaveResponse` (`refund_due`, `tax_due`), `TaxCalendarSummaryResponse` (`start_year`, `end_year`).
**AC:**
- [ ] לכל שדה: להחליט required (לא-null) או optional (ניתן להיעדר) — ולתקן את ה-Pydantic model בהתאם
- [ ] codegen מייצר טיפוסים עקביים (אין `required` + `| null` יחד)

#### 75. `period` — regex validation רק ב-1 מתוך 19 schemas
**בעיה:** השדה `period` (פורמט `YYYY-MM`) מופיע ב-19 schemas, אבל ה-regex `^\d{4}-(0[1-9]|1[0-2])$` קיים רק ב-`AdvancePaymentCreateRequest`. שאר ה-input schemas (`VatWorkItemCreateRequest`, `ChargeCreateRequest`) מקבלים `period` חופשי — אפשר לשלוח `period: "garbage"` ל-VAT work item והוא יתקבל.
**AC:**
- [ ] ה-regex מוחל על כל ה-Create/input schemas שמקבלים `period`
- [ ] עדיף: type משותף (`PeriodStr`) שמרכז את ה-validation במקום אחד
- [ ] בדיקה: `period` לא חוקי מוחזר עם `422`

---

## בוצע כבר - שאר הפריטים

### Security / Auth

#### 1. Security global default ✅ בוצע
**בעיה:** ה-`security` הגלובלי ב-spec ריק (`[]`). האבטחה הוגדרה ידנית על כל endpoint. endpoint חדש שנכתב בלי `security` field היה פתוח ללא אימות.
**AC:**
- [x] `HTTPBearer` מוגדר כ-`security` ברמת ה-spec הגלובלית
- [x] endpoints ציבוריים (login, refresh, forgot/reset-password, health, `/sign/{token}`) מוגדרים במפורש כ-`security: []`
- [x] בדיקה: endpoint חדש ללא הגדרת security מקבל אימות אוטומטית

**מה בוצע:**
- `build_openapi()` ב-[app/core/openapi.py](../backend/app/core/openapi.py) קובע `security: [{HTTPBearer: []}]` גלובלי + `securitySchemes.HTTPBearer`, ומחיל `security: []` על ה-public endpoints בלבד. מחובר ל-app דרך `custom_openapi` ב-[app/main.py](../backend/app/main.py).
- allowlist יחיד ב-[app/core/public_endpoints.py](../backend/app/core/public_endpoints.py) (source of truth, משותף ל-spec ולטסט).
- **אכיפת runtime** (לא רק תיעוד): [tests/core/test_endpoint_auth_guard.py](../backend/tests/core/test_endpoint_auth_guard.py) נכשל אם endpoint מחוץ ל-allowlist חסר `get_current_user`/`require_role` (רקורסיבי — תופס גם `require_role` מקונן). בנוסף מוודא ש-public ops מסומנים `security: []` ושאף non-public לא.
- `logout` עבר לדרוש אימות (הוסר מה-allowlist + קיבל auth dependency).
- `openapi.json` יוצא מחדש ו-`check_contract_sync` ירוק. תיעוד: כלל ב-[docs/architecture/security.md](architecture/security.md).

### Pagination / Audit

#### 24. Pagination — איחוד offset/limit ✅ בוצע
**בעיה:** שני endpoints השתמשו ב-`offset`/`limit` במקום `page`/`page_size`.
**AC:**
- [x] `GET /vat/work-items/{item_id}/audit` עובר ל-`page`/`page_size`
- [x] `GET /audit/{entity_type}/{entity_id}` עובר ל-`page`/`page_size`
- [x] ה-response envelope תואם לשאר ה-paginated responses

#### 55. שני audit endpoints עם shapes שונות ✅ בוצע
**בעיה:** `VatAuditTrailResponse` היה `{items, total}` ו-`EntityAuditTrailResponse` היה `{items, total, limit, offset}`.
**AC:**
- [x] envelope אחיד לשני ה-audit endpoints: שניהם מחזירים `items`, `total`, `page`, `page_size`.

### Tasks

#### 20. `GET /api/v1/tasks` עם filter לפי ישות ✅ בוצע
**בעיה:** אי אפשר לשלוף משימות של VAT work item / דוח שנתי ספציפי.
**AC:**
- [x] תמיכה ב-`?source_domain=&source_id=` כ-query params
- [x] השמות הקיימים `source_domain`/`source_id` נשארו השמות הקנוניים, זהים ל-`TaskResponse`

**בוצע:** אומת בקוד:
- `backend/app/tasks/api/routes.py` — `list_tasks()` מקבל `source_domain: WorkQueueSourceType | None = Query(None)` ו-`source_id: int | None = Query(None)` ומעביר אותם לשירות.
- `backend/app/tasks/schemas/task.py` — `TaskResponse` מחזיר `source_domain` ו-`source_id` באותם שמות.
- `backend/app/tasks/services/task_service.py` — `TaskService.list()` מקבל ומעביר את שני ה-filters.
- `backend/app/tasks/repositories/task_repository.py` — `_apply_filters()` מוסיף `Task.source_domain == source_domain` ו-`Task.source_id == source_id`; `list_active()` מחזיר רק non-deleted עם pagination.
- `backend/tests/tasks/test_task_service.py` — `test_list_filter_by_source_domain` מאמת שה-list מסנן לפי source domain.
- `backend/tests/tasks/test_task_api.py` — create/update tests מאמתים את contract של `source_domain`/`source_id` ואת החזרת השדות ב-response.
- `backend/openapi.json` — `GET /api/v1/tasks` מתעד `source_domain` ו-`source_id` כ-query params.

### Annual Reports

#### 47. 🔍 `status` vs `transition` ב-annual-reports ✅ בוצע
**בעיה:** שני endpoints למעבר סטטוס, schemas שונים (`StatusTransitionRequest` vs `StageTransitionRequest`).
**AC:**
- [x] ההבחנה תועדה: `transition` הוא stage shortcut שממופה ל-status; `status` הוא מעבר סטטוס ישיר.
- [x] הקוד מממש את ההבחנה דרך `STAGE_TO_STATUS` וקריאה ל-`transition_status()`.

**בוצע:** אומת בקוד ובתיעוד:
- `docs/flows/02-annual-report-status-transition.md` — מתעד ש-stage transitions ממופים ל-status דרך `STAGE_TO_STATUS`, ו-`transition_stage()` קורא ל-`transition_status()`.
- `backend/app/annual_reports/api/annual_report_stage_transition.py` — `POST /annual-reports/{report_id}/transition` מקבל `StageTransitionRequest`.
- `backend/app/annual_reports/api/annual_report_status.py` — `POST /annual-reports/{report_id}/status` מקבל `StatusTransitionRequest`.
- `backend/app/annual_reports/services/constants.py` — `STAGE_TO_STATUS` מגדיר את מפת stage→status.
- `backend/app/annual_reports/services/status_service.py` — `transition_stage()` ממפה `to_stage` ל-status וקורא ל-`transition_status()`.
- `backend/tests/annual_reports/api/test_annual_report_stage_transition.py` — בדיקת API ל-stage transition.

### Error / Response Contracts

#### 25. 404 responses ✅ בוצע
**בעיה:** ~120 endpoints שמקבלים ID param לא תיעדו `404` ב-spec (אף שהקוד מחזיר).
**AC:**
- [x] כל endpoint עם path param של ID כולל `404` ב-responses
- [x] codegen מייצר טיפול ב-404 (דורש הרצת codegen ב-frontend — לא בוצע כאן)

**בוצע:** sweep מלא — כל 125 ה-endpoints עם path param של ID מתעדים כעת `404` עם `ErrorEnvelope` דרך `not_found_response()`. נוסף טסט רגרסיה `backend/tests/core/test_openapi_not_found_docs.py` שדורס על `app.openapi()` ונכשל אם endpoint עם ID param חסר `404` (ללא חריגים — `ALLOWED_NO_404` ריק). חריגים מכוונים: `/reminders/` list+create, `/advance-payments/overview*`, `/clients/conflict/{id_number}` — ללא זהות-ישות אמיתית.

#### 43. 🔍 metadata נוסף לא עקבי ✅ בוצע
**בעיה:** `stats`/`counters`/`summary` רק בחלק מה-responses.
**AC:**
- [x] בירור: כל 5 בלוקי ה-metadata (BinderListResponse.counters, ChargeListResponse.stats, ClientRecordListResponse.stats, WorkQueueListResponse.summary, TaxCalendarGroupListResponse.summary) חיוניים למסך ונצרכים ב-UI — אף אחד לא הוסר.
- [x] מדיניות תועדה ב-[api-contracts.md](architecture/api-contracts.md): aggregate metadata רק כשה-UI צריך; שם מועדף ל-API חדשים = `summary`; `stats`/`counters` קיימים נשארים כשמוצדקים. לא נכפה `summary` על כל list, לא הוסף `summary: {}` ריק, ולא שונה שם לשם טוהר.

#### 44. `sort_order` vs `order` ✅ בוצע
**בעיה:** שני שמות לאותו param.
**החלטה:** הסטנדרט הוא **`order`** (לא `sort_order`), לפי source-of-truth [api-contracts.md](architecture/api-contracts.md) — list endpoints חייבים `page,page_size,sort_by,order`; aliases כמו `sort_dir`/`sort_order` חייבים להגר ל-`order`. ה-AC המקורי ("שם אחיד `sort_order`") היה הפוך ותוקן.
**AC:**
- [x] שם אחיד אחד (`order`) בכל ה-endpoints: annual-reports, binders, correspondence כבר השתמשו ב-`order`. הוסף guard `Literal["asc","desc"]` ל-binders.
- [x] `clients` (`GET /clients`, `GET /clients/sidebar`) הוגר מ-`sort_order` ל-`order` (route+service+frontend+URL state), ללא alias תאימות.
- [x] ה-frontend מעודכן (clients types/contracts/hooks/filters; binders כבר שלח `order`).

#### 45. POST מחזיר 200 במקום 201 ✅ בוצע
**בעיה:** 4 endpoints חשודים כיוצרים משאב ומחזירים 200.
**AC:**
- [x] בירור לכל endpoint. מסקנה: **כל 4 נשארים 200** — אף אחד לא מחזיר ייצוג של משאב חדש שניתן לאחזר.

| Endpoint | מצב | יוצר משאב אחזיר? | החלטה | סיבה |
|---|---|---|---|---|
| `POST /clients/import` | 200 | לא — דו"ח batch | **200** | מחזיר סיכום `{created, total_rows, errors[]}`; יכול ליצור 0 (כל השורות נכשלות); partial-success; idempotency-guarded; לא ייצוג משאב |
| `POST /charges/bulk-action` | 200 | לא — פעולה | **200** | פעולה על charges קיימים, idempotent |
| `POST /annual-reports/{id}/deadline` | 200 | לא — עדכון | **200** | מעדכן deadline על דוח קיים |
| `POST /auth/forgot-password` | 200 | לא (security-neutral) | **200** | מניעת user enumeration; הלקוח לא צורך משאב |

#### 54. `GET /annual-reports/{id}/charges` — response schema ריק ✅ בוצע
**בעיה:** מחזיר `{}` במקום schema מוגדר.
**AC:**
- [x] schema מפורש לתגובה

**בוצע:** `GET /api/v1/annual-reports/{report_id}/charges` מצהיר `response_model=PaginatedResponse[ChargeResponse]`, וה-OpenAPI מייצר `PaginatedResponse_ChargeResponse_`.

### DTO / Schema Cleanup

#### 34. DTOs שמנים ב-double-duty (list + detail) ✅ בוצע
**בעיה:** ארבעה schemas החזירו את *כל* השדות גם ברשימה. טעינת 50 פריטים = 50× כל השדות, כולל שדות שרלוונטיים רק ל-detail.
**מושפעים:**
- `VatWorkItemResponse` (36 שדות)
- `ClientRecordResponse` (27 שדות)
- `ChargeResponse` (23 שדות)
- `NotificationResponse` (24 שדות)

**AC:**
- [x] DTO רזה (`XxxListItem`) לכל אחד מה-4: `VatWorkItemListItem`, `ClientRecordListItem`, `ChargeListItem`, `NotificationListItem`
- [x] endpoints של LIST עוברים ל-DTO הרזה (vat work-items + grouped + by-client, clients, charges, notifications)
- [x] ה-DTO השמן נשאר רק ב-`GET /{id}` — וב-notifications נוסף `GET /notifications/{id}` חדש (לא היה detail endpoint)
- [x] מדידה: [performance/list-dto-payloads.md](performance/list-dto-payloads.md) (VAT −56.9%, clients −66.0%, charges −32.7%, notifications −31.6%)

**הושלם** (2026-06-11). חוק contract חדש ב-[architecture/api-contracts.md](architecture/api-contracts.md). תיעוד דומיינים עודכן (vat/charges/clients/notifications). הערה: notifications list שומר `content_snapshot`/`subject_snapshot`/`business_name` כי ה-bell drawer/tab מציגים preview inline.

#### 35. 🔍 שדות כפולים חשודים ב-`AnnualReportDetailResponse` ✅ בוצע
**בעיה:** שלושה זוגות שנראו שרידי מיגרציה — `refund_due`+`tax_refund_amount`, `tax_due`+`tax_due_amount`, `assessment_amount`+`final_balance`.
**ממצא:** רק זוג אחד אמיתי. `tax_refund_amount`/`tax_due_amount` היו עותקי float לא-מוצגים של עמודות ה-DB הקנוניות `refund_due`/`tax_due`. `assessment_amount` (קלט שומת רשות) ו-`final_balance` (מחושב: `tax_after_credits − advances_paid`) אינם כפילות.
**AC:**
- [x] canonical: `refund_due` על פני `tax_refund_amount`, `tax_due` על פני `tax_due_amount` (עמודות DB, בשימוש ברחבי status/list/history)
- [x] השדות הכפולים הוסרו (לא deprecated — entry-point.md אוסר legacy compat ללא בקשה; השדות לא הוצגו ב-UI)
- [x] אין שבירה ב-UI (typecheck+lint frontend עוברים; הצריכה היחידה הייתה merge ב-`useReportMutations`)

#### 35b. עותקי `tax_refund_amount`/`tax_due_amount` ב-`ReportDetailResponse` ✅ בוצע
**בעיה:** אותם עותקי float מתים שהוסרו מ-`AnnualReportDetailResponse` (פריט 35) עדיין היו קיימים ב-`ReportDetailResponse` — endpoint נפרד `GET/PATCH /annual-reports/{id}/details` (`annual_report_detail.py` + `_enrich_detail_response`).
**AC:**
- [x] אומת: ב-frontend השדות הופיעו רק ב-interface `ReportDetailResponse` (`contracts.ts`) וב-`generated.ts` — לא נצרכו בשום קומפוננטה
- [x] הוסרו מ-`ReportDetailResponse` (`annual_report_detail.py` schema). `_enrich_detail_response` נמחק כולו — מטרתו היחידה הייתה הצבת שני השדות, כך שגם query מיותר ל-DB נחסך
- [x] ה-frontend manual contract של `ReportDetailResponse` נוקה גם משדות credit-point מחושבים (`credit_points`, `pension_credit_points`, `life_insurance_credit_points`, `tuition_credit_points`) שאינם חוזרים מ-`GET/PATCH /details`; הערכים המחושבים זמינים רק דרך `AnnualReportDetailResponse.tax_calculation`
- [x] frontend נקי (0 הפניות); generated types + OpenAPI מעודכנים; tests עוברים (105)

#### 36. קיבוץ חישובי מס ב-`AnnualReportDetailResponse` ✅ בוצע
**בעיה:** חישובי מס שטוחים מעורבבים עם זהות/סטטוס/מטא-דאטה.
**AC:**
- [x] כל פלטי החישוב (11 שדות) מקובצים ל-`tax_calculation` (`AnnualReportTaxCalculationResponse`); קלטי ניכוי שמוזנים ידנית ועמודות outcome נשארו flat
- [x] ה-frontend מעודכן למבנה החדש (`report.tax_calculation.*`)
- [x] ה-schema הראשי שטוח-יותר; מבנה היררכי

#### 37. 🔍 DTO רזה ל-`AnnualReportResponse` (31 שדות) ברשימה ✅ בוצע
**בעיה:** `GET /annual-reports` החזיר 31 שדות לכל פריט. (אין `AnnualReportCard` לדומיין זה — ההפניה המקורית הייתה לדומיין אחר.)
**AC:**
- [x] נוצר `AnnualReportListItem` רזה (15 שדות שה-UI באמת מציג); list/overdue/season/client endpoints עברו אליו, detail נשאר `AnnualReportDetailResponse`
- [x] mapper נפרד `_to_list_items` מדלג על חישוב actions/transitions היקר
- [x] ה-frontend list contract/types מעודכנים; regression guard מוודא הפרדת list/detail

#### 38. שמות schemas לא עקביים ✅ בוצע
**בעיה:** רוב הדומיינים `XxxCreateRequest`, אבל `CreateClientRequest` הפוך, ו-`BinderIntakePatchRequest` חורג.
**AC:**
- [x] `CreateClientRequest` → `ClientOnboardingRequest` (היעד `ClientCreateRequest` כבר תפוס ע"י מחלקה אחרת; השם החדש מדויק כי זה composite של client + business)
- [x] `BinderIntakePatchRequest` → `BinderIntakeUpdateRequest`
- [x] כל ה-frontend references מעודכנים (ללא alias תאימות)

#### 39. Create = Update (8 schemas זהים) ✅ בוצע
**בעיה:** schemas של Create ו-Update היו זהים, כך ש-PATCH לא partial בפועל — שולחים את כל השדות כמו PUT.
**מושפעים:** AuthorityContact, Correspondence, ExpenseLine, IncomeLine, Task, Business (Create/Update כל אחד).
**AC:**
- [x] בכל זוג: Update הופך לכל-שדות-אופציונליים אמיתי (`exclude_unset=True`, הוסר None-dropping פנימי)
- [x] תיעוד: PATCH עם שדה בודד לא מאפס שאר השדות
- [x] בדיקה לכל זוג

#### 40. 🔍 שדות נסתרים ב-`ClientUpdateRequest` ✅ בוצע
**בעיה:** `status`, `annual_revenue`, `advance_rate_updated_at` קיימים ב-Update אבל לא ב-Create.
**AC:**
- [x] בירור: `status` + `annual_revenue` נשארים user-editable; `advance_rate_updated_at` הפך server-owned (מתועד ב-[update-request-conventions.md](architecture/update-request-conventions.md))
- [x] ה-frontend חושף `status`/`annual_revenue`; `advance_rate_updated_at` הוסר מ-update payload (frontend + taxProfile) ונשאר read-only ב-response

#### 41. 🔍 UpdateRequest ללא validation ✅ בוצע
**בעיה:** 30 Update schemas ללא `required` ולא הגבלות. ניתן לשלוח PATCH ריק `{}` ולקבל 200.
**AC:**
- [x] בירור: PATCH ריק `{}` → **422** (validation ב-Pydantic/FastAPI, לא 400, כדי למנוע כפילות endpoint)
- [x] בירור: `title: ""` / whitespace ב-`TaskUpdateRequest` → 422 (`NonBlankStr`)
- [x] התנהגות מתועדת + validation בהתאם

**מה בוצע (38–41):**
- `NonEmptyUpdateMixin` ([app/core/schemas/validation.py](../backend/app/core/schemas/validation.py)) — `extra="forbid"` (שדה לא מוכר → 422) + דחיית `{}` ריק → 422. הוחל על כל 15 ה-`*UpdateRequest` (14 קיימים + `BinderIntakeUpdateRequest` ששמו שונה).
- `NonBlankStr` ([app/core/api_types.py](../backend/app/core/api_types.py)) — strip + `min_length=1`, על שדות טקסט עסקיים-נדרשים: `title`, `subject`, `business_name`, `name`, `full_name` (client+user), `note`.
- explicit-null נדחה (422) על שדות non-nullable לפי עמודת המודל (לא לפי Create-optionality); שדות nullable אמיתיים מתנקים ע"י null.
- חריגים מתועדים (single-payload required): `EntityNoteUpdateRequest.note`, `AnnexDataUpdateRequest.data`. `DeadlineUpdateRequest` הפך partial (POST, deadline_type אופציונלי).
- VAT invoice update עבר ל-`exclude_unset` semantics מלא + ולידציה של ה-counterparty pair האפקטיבי (merged) בשירות.
- תיעוד: [docs/architecture/update-request-conventions.md](architecture/update-request-conventions.md) (רשום ב-documentation-map). OpenAPI + frontend `generated.ts` יוצאו מחדש.
- בדיקות: סוויטת backend מלאה ירוקה; frontend typecheck/lint/build ירוקים.

#### 46. `updated_at` חסר על Response schemas ✅ בוצע
**בעיה:** קיים `created_at` בכל, אבל `updated_at` רק בחלק.
**מצב:** כל 6 ה-schemas שב-AC מחזיקים כעת `updated_at` אמיתי. לא מזייפים מ-`created_at` (לא ב-runtime ולא ב-migration): העמודה nullable, מתחילה NULL, ומתמלאת רק ב-update אמיתי דרך `onupdate`. כל מודל קיבל עמודה + Alembic migration (0002–0005) + audit של נתיבי העדכון + בדיקות, ו-OpenAPI/`generated.ts` יוצאו מחדש.
**AC:**
- [x] `ReminderResponse` — כבר כלל `updated_at` (no-op).
- [x] `BusinessResponse` — נוסף `updated_at` (העמודה קיימת במודל, nullable + `onupdate`). additive.
- [x] `ChargeResponse` — נוספה עמודה `updated_at` (nullable, `onupdate=utcnow`) + migration 0002. מתעדכן ב-issue/pay/cancel/soft-delete.
- [x] `BinderResponse` — נוספה עמודה `updated_at` (nullable, `onupdate=utcnow`) + migration 0003. מתעדכן בשינויי lifecycle/handover/capacity/soft-delete.
- [x] `SignatureRequestResponse` — נוספה עמודה `updated_at` (nullable, `onupdate=utcnow`) + migration 0004. מתעדכן ב-send/sign/decline/cancel/expire/soft-delete. `SignatureAuditEvent` נשאר append-only ללא `updated_at`.
- [x] `CorrespondenceResponse` — **החלטת דומיין: correspondence הוא mutable**. ה-PATCH path אמיתי ונשאר; נוספה עמודה `updated_at` (nullable, `onupdate=utcnow_aware`) + migration 0005, ה-docstring המיושן ("No updated_at"/immutable) ו-[docs/domains/communications.md](domains/communications.md) תוקנו.

### Cross-System Consistency

#### 71. `history` vs `audit` ✅ בוצע
**סטטוס:** `audit` הוא הדפוס הקנוני לנתיבי audit trail; נתיבי binders/annual-reports הועברו ל-`/audit`.
**AC:**
- [x] בחירת דפוס אחד + יישום בכל הדומיינים.
