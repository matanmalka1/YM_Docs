# API TODO — מפורט עם Acceptance Criteria
_מבוסס על gap analysis מ-OpenAPI spec | יוני 2026_

**מקרא:** 🔍 = דורש בירור/החלטה לפני תיקון · AC = Acceptance Criteria (מתי המשימה גמורה)

---

## 🔴 עדיפות גבוהה

### 1. Security global default ✅ בוצע
**בעיה:** ה-`security` הגלובלי ב-spec ריק (`[]`). האבטחה מוגדרת ידנית על כל endpoint. endpoint חדש שנכתב בלי `security` field יהיה פתוח ללא אימות — וזה לא נראה לעין בקוד.
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

### 2. `PATCH /api/v1/vat/work-items/{item_id}`
**בעיה:** אין דרך לעדכן work item אחרי יצירה (סטטוס, הערות, שדות פנימיים).
**AC:**
- [ ] endpoint מקבל `VatWorkItemUpdateRequest` עם שדות אופציונליים
- [ ] עדכון חלקי (שדה בודד) לא מאפס שדות אחרים
- [ ] מחזיר `VatWorkItemResponse` מעודכן + `404` אם לא קיים

### 3. `DELETE /api/v1/vat/work-items/{item_id}`
**בעיה:** אין מחיקת work item.
**AC:**
- [ ] מחיקה רכה (soft delete) עקבית עם שאר הדומיינים
- [ ] מחזיר `204` בהצלחה, `404` אם לא קיים
- [ ] פריט מחוק לא מופיע ב-LIST כברירת מחדל

### 4. `POST /api/v1/vat/work-items/bulk-transition`
**בעיה:** אין שינוי סטטוס מרובה — צריך לעבור פריט-פריט.
**AC:**
- [ ] מקבל רשימת IDs + transition יעד
- [ ] מחזיר סיכום הצלחות/כשלונות לכל פריט (כמו `BulkChargeActionResponse`)
- [ ] transition לא חוקי לפריט בודד לא מפיל את כל ה-batch

### 5. `PATCH /api/v1/charges/{charge_id}`
**בעיה:** charge שנוצר לא ניתן לעריכה (סכום, תיאור, תאריך).
**AC:**
- [ ] עדכון מותר רק בסטטוס `draft` (בירור 🔍: האם לאפשר גם ב-`issued`?)
- [ ] מחזיר `ChargeResponse` מעודכן + `409` אם הסטטוס לא מאפשר עריכה

### 6. `PATCH /api/v1/notifications/{notification_id}` — mark read/unread
**בעיה:** אין endpoint לסימון notification כנקרא. ה-UI כנראה עוקף את זה בצד הלקוח.
**AC:**
- [ ] מאפשר עדכון סטטוס קריאה
- [ ] מחזיר ה-notification מעודכן + `404` אם לא קיים

### 7. `POST /api/v1/notifications/mark-all-read`
**בעיה:** אין סימון גורף.
**AC:**
- [ ] מסמן את כל ה-notifications של המשתמש כנקראו
- [ ] מחזיר מספר הפריטים שעודכנו

### 8. `DELETE /api/v1/notifications/{notification_id}` — dismiss
**בעיה:** אין הסרה/dismiss.
**AC:**
- [ ] מסיר notification מהתצוגה
- [ ] מחזיר `204` + `404` אם לא קיים

### 9. `PATCH /api/v1/reminders/{reminder_id}`
**בעיה:** אין עדכון תזכורת (תאריך, טקסט).
**AC:**
- [ ] עדכון חלקי של תזכורת בסטטוס `scheduled`
- [ ] מחזיר `409` אם התזכורת כבר `fired`/`canceled`

### 10. `DELETE /api/v1/reminders/{reminder_id}`
**בעיה:** אין מחיקה — רק cancel.
**AC:**
- [ ] בירור 🔍: האם DELETE נחוץ בנוסף ל-cancel, או ש-cancel מספיק? אם כן — `204` + soft delete

---

## 🟠 עדיפות בינונית

### 11. `GET /api/v1/documents/client/{client_record_id}/{document_id}`
**בעיה:** אפשר לרשום, להעלות, למחוק ולהחליף מסמך — אבל אין שליפת מסמך בודד.
**AC:**
- [ ] מחזיר metadata של מסמך יחיד + `404`

### 12. `PATCH /api/v1/documents/client/{client_record_id}/{document_id}`
**בעיה:** אין עדכון metadata (שם, סוג, תגיות) בלי להחליף את הקובץ.
**AC:**
- [ ] עדכון metadata בלבד, ללא העלאה מחדש
- [ ] לא משנה את גרסת הקובץ

### 13. `GET /api/v1/documents/binder/{binder_id}`
**בעיה:** יש `documents/client/{id}` ו-`documents/annual-report/{id}` אבל לא לפי binder.
**AC:**
- [ ] מחזיר מסמכים המשויכים ל-binder, באותו envelope כמו שאר רשימות המסמכים

### 14–18. ניהול Tax Calendar (settings)
**בעיה:** settings הם read-only + bootstrap בלבד. אין יצירה/עדכון/מחיקה של חוקים ו-entries.
**AC:**
- [ ] `PATCH /settings/tax-calendar/rules/{rule_id}` — עדכון חוק
- [ ] `POST /settings/tax-calendar/rules` — יצירת חוק
- [ ] `DELETE /settings/tax-calendar/rules/{rule_id}` — מחיקה
- [ ] `PATCH /settings/tax-calendar/entries/{entry_id}` — עדכון entry
- [ ] `DELETE /settings/tax-calendar/entries/{entry_id}` — מחיקה
- [ ] בירור 🔍: עדכון חוק — האם משפיע רטרואקטיבית על entries קיימים או רק עתידיים?

### 19. `GET /api/v1/clients/{client_record_id}/tasks`
**בעיה:** אין שליפת משימות לפי לקוח — חובה לסנן ידנית.
**AC:**
- [ ] מחזיר משימות מסוננות ל-client_record_id, עם pagination אחיד

### 20. `GET /api/v1/tasks` עם filter לפי ישות
**בעיה:** אי אפשר לשלוף משימות של VAT work item / דוח שנתי ספציפי.
**AC:**
- [ ] תמיכה ב-`?source_domain=&source_id=` כ-query params
- [ ] בירור 🔍: לאחד עם השמות הקיימים `source_domain`/`source_id` שכבר ב-`TaskResponse`

### 21. `POST /api/v1/binders` + `PATCH /api/v1/binders/{binder_id}`
**בעיה:** binders פועלים רק דרך transitions — אין POST/PATCH ישיר.
**AC:**
- [ ] בירור 🔍: האם המודל הנוכחי (receive-only) מכוון? אם כן — לתעד ולסגור. אם לא — להוסיף POST/PATCH

### 22. `POST /api/v1/binders/{binder_id}/restore`
**בעיה:** restore קיים ב-clients אבל לא ב-binders אחרי DELETE.
**AC:**
- [ ] משחזר binder מחוק, עקבי עם `clients/{id}/restore`

### 23. `PATCH /api/v1/annual-reports/{report_id}`
**בעיה:** אין PATCH ראשי — רק `PATCH /{id}/details`.
**AC:**
- [ ] בירור 🔍: האם `/details` מספיק או שצריך wrapper ראשי? להחליט ולתעד

### 24. Pagination — איחוד offset/limit
**בעיה:** שני endpoints משתמשים ב-`offset`/`limit` במקום `page`/`page_size`.
**AC:**
- [x] `GET /vat/work-items/{item_id}/audit` עובר ל-`page`/`page_size`
- [x] `GET /audit/{entity_type}/{entity_id}` עובר ל-`page`/`page_size`
- [x] ה-response envelope תואם לשאר ה-paginated responses

### 25. 404 responses
**בעיה:** ~120 endpoints שמקבלים ID param לא מתעדים `404` ב-spec (אף שהקוד מחזיר).
**AC:**
- [x] כל endpoint עם path param של ID כולל `404` ב-responses
- [x] codegen מייצר טיפול ב-404 (דורש הרצת codegen ב-frontend — לא בוצע כאן)

**בוצע:** sweep מלא — כל 125 ה-endpoints עם path param של ID מתעדים כעת `404` עם `ErrorEnvelope` דרך `not_found_response()`. נוסף טסט רגרסיה `backend/tests/core/test_openapi_not_found_docs.py` שדורף על `app.openapi()` ונכשל אם endpoint עם ID param חסר `404` (ללא חריגים — `ALLOWED_NO_404` ריק). חריגים מכוונים: `/reminders/` list+create, `/advance-payments/overview*`, `/clients/conflict/{id_number}` — ללא זהות-ישות אמיתית.

---

## 🟡 עדיפות נמוכה

### 26–27. Bulk operations ל-clients
**AC:**
- [ ] `POST /clients/bulk-patch` — עדכון מרובה, מחזיר סיכום הצלחות/כשלונות
- [ ] `POST /clients/bulk-delete` — מחיקה/ארכוב מרובה

### 28. `GET /api/v1/signature-requests` — LIST כללי
**בעיה:** קיים רק `/pending`.
**AC:**
- [ ] LIST עם filter על סטטוס (כולל `pending` כמקרה פרטי)

### 29. `POST /api/v1/signature-requests/{request_id}/cancel` — ישיר
**בעיה:** cancel קיים רק דרך `/clients/{id}/signature-requests/{id}/cancel`.
**AC:**
- [ ] cancel ישיר ללא צורך ב-client context, עקבי עם שאר ה-cancel

### 30. `POST /api/v1/charges/{charge_id}/restore`
**AC:**
- [ ] משחזר charge מבוטל/מחוק (בירור 🔍: מאיזה סטטוס ניתן לשחזר?)

### 31. `DELETE /api/v1/users/{user_id}`
**בעיה:** אין DELETE — רק deactivate.
**AC:**
- [ ] בירור 🔍: האם soft delete בלבד מכוון (audit/compliance)? אם כן — לתעד ולסגור

### 32–33. Bulk operations ל-tasks
**AC:**
- [ ] `POST /tasks/bulk-complete` — השלמה מרובה
- [ ] `POST /tasks/bulk-assign` — שיוך מחדש מרובה

---

## 🟧 DTOs שמנים ו-Over-fetching

### 34. DTOs שמנים ב-double-duty (list + detail)
**בעיה:** ארבעה schemas מחזירים את *כל* השדות גם ברשימה. טעינת 50 פריטים = 50× כל השדות, כולל שדות שרלוונטיים רק ל-detail. הפתרון כבר קיים במערכת: `AnnualReportCard` (5 שדות) לצד `AnnualReportDetailResponse` (52).
**מושפעים:**
- `VatWorkItemResponse` (36 שדות) — רשימה מחזירה `override_justification`, `submission_reference`, `filed_by_name`, `statutory_deadline`, `extended_deadline`
- `ClientRecordResponse` (27), `ChargeResponse` (23), `NotificationResponse` (24)
**AC:**
- [ ] DTO רזה (`XxxCard` / `XxxListItem`) לכל אחד מה-4, עם רק השדות שמוצגים בטבלה
- [ ] endpoints של LIST עוברים ל-DTO הרזה
- [ ] ה-DTO השמן נשאר רק ב-`GET /{id}`
- [ ] מדידה: payload של LIST קטֵן (לתעד before/after)

### 35. 🔍 שדות כפולים חשודים ב-`AnnualReportDetailResponse`
**בעיה:** שלושה זוגות שנראים שרידי מיגרציה — `refund_due`+`tax_refund_amount`, `tax_due`+`tax_due_amount`, `assessment_amount`+`final_balance`.
**AC:**
- [ ] בירור: איזה מכל זוג canonical? לבדוק שימוש ב-frontend
- [ ] השדה המיותר מסומן deprecated, ואז מוסר
- [ ] אין שבירה ב-UI (בדיקה לפני הסרה)

### 36. קיבוץ חישובי מס ב-`AnnualReportDetailResponse`
**בעיה:** 18 מתוך 52 השדות הם חישובי מס שטוחים. ה-schema מערבב 5 תחומי אחריות.
**AC:**
- [ ] חישובי המס מקובצים ל-sub-object `tax_calculation: { ... }`
- [ ] ה-frontend מעודכן למבנה החדש
- [ ] ה-schema הראשי יורד מ-52 שדות שטוחים למבנה היררכי

### 37. 🔍 DTO רזה ל-`AnnualReportResponse` (31 שדות) ברשימה
**בעיה:** `GET /annual-reports` מחזיר 31 שדות לכל פריט, אף ש-`AnnualReportCard` (5 שדות) קיים.
**AC:**
- [ ] בירור: מדוע הרשימה לא משתמשת ב-Card? אם מספיק — לעבור אליו

---

## 🔵 בעיות DTO ו-Schema

### 38. שמות schemas לא עקביים
**בעיה:** רוב הדומיינים `XxxCreateRequest`, אבל `CreateClientRequest` הפוך, ו-`BinderIntakePatchRequest` חורג.
**AC:**
- [ ] `CreateClientRequest` → `ClientCreateRequest`
- [ ] `BinderIntakePatchRequest` → `BinderIntakeUpdateRequest`
- [ ] כל ה-frontend references מעודכנים

### 39. Create = Update (8 schemas זהים)
**בעיה:** schemas של Create ו-Update זהים, כך ש-PATCH לא partial בפועל — שולחים את כל השדות כמו PUT.
**מושפעים:** AuthorityContact, Correspondence, ExpenseLine, IncomeLine, Task, Business (Create/Update כל אחד).
**AC:**
- [ ] בכל זוג: Update הופך לכל-שדות-אופציונליים אמיתי
- [ ] תיעוד: PATCH עם שדה בודד לא מאפס שאר השדות
- [ ] בדיקה לכל זוג

### 40. 🔍 שדות נסתרים ב-`ClientUpdateRequest`
**בעיה:** `status`, `annual_revenue`, `advance_rate_updated_at` קיימים ב-Update אבל לא ב-Create.
**AC:**
- [ ] בירור: מדוע השדות רק ב-Update? לתעד במפורש
- [ ] ה-frontend חושף אותם בטופס העריכה (לוודא ש-`status` לא נשמט)

### 41. 🔍 UpdateRequest ללא validation
**בעיה:** 30 Update schemas ללא `required` ולא הגבלות. ניתן לשלוח PATCH ריק `{}` ולקבל 200.
**AC:**
- [ ] בירור: מה ההתנהגות הרצויה ל-PATCH ריק? (no-op / 400?)
- [ ] בירור: `title: ""` ב-`TaskUpdateRequest` — חוקי?
- [ ] התנהגות מתועדת + validation בהתאם

---

## 🟣 בעיות Response ו-Envelope

### 42. Response envelopes לא עקביים
**בעיה:** 4 וריאנטים שונים של רשימה.
**AC:**
- [ ] `VatWorkItemListResponse` — מוסיף `page`, `page_size`
- [ ] `VatInvoiceListResponse` — מוסיף `page`, `page_size`, `total`
- [ ] `BinderIntakeListResponse` — `intakes` → `items`
- [ ] `CorrespondenceListResponse` — `total_pages` מוסר או מתווסף לכולם (החלטה אחת)

### 43. 🔍 metadata נוסף לא עקבי
**בעיה:** `stats`/`counters`/`summary` רק בחלק מה-responses.
**AC:**
- [ ] בירור: מבנה metadata אחיד לכל LIST שצריך, או הסרה מאלה שלא צריך

### 44. `sort_order` vs `order`
**בעיה:** שני שמות לאותו param.
**AC:**
- [ ] שם אחיד אחד (`sort_order`) בכל ה-endpoints: annual-reports, binders, correspondence
- [ ] ה-frontend מעודכן

### 45. POST מחזיר 200 במקום 201
**בעיה:** 4 endpoints יוצרים משאב ומחזירים 200.
**AC:**
- [ ] `POST /clients/import`, `/charges/bulk-action`, `/annual-reports/{id}/deadline`, `/auth/forgot-password` — בירור 🔍 לכל אחד: האם באמת יוצר משאב? אם כן → 201

### 46. `updated_at` חסר על Response schemas
**בעיה:** קיים `created_at` בכל, אבל `updated_at` רק בחלק.
**AC:**
- [ ] להוסיף `updated_at` ל: BinderResponse, ChargeResponse, CorrespondenceResponse, BusinessResponse, SignatureRequestResponse, ReminderResponse

---

## 🟤 כפילויות ו-Versioning

### 47. 🔍 `status` vs `transition` ב-annual-reports
**בעיה:** שני endpoints למעבר סטטוס, schemas שונים (`StatusTransitionRequest` vs `StageTransitionRequest`).
**AC:**
- [ ] בירור: מה ההבדל? אם עודף — להסיר אחד. אם לא — לתעד את ההבחנה בין status ל-stage

### 48. 🔍 `GET /annual-reports` vs `GET /clients/{id}/annual-reports`
**בעיה:** הראשון כבר מסנן לפי `client_record_id`, אז השני מיותר.
**AC:**
- [ ] בירור: להסיר את השני או לתעד מדוע נחוצים שניהם

### 49. `GET /reports/annual-reports` — שם מבלבל
**בעיה:** מחזיר `AnnualReportStatusReportResponse` (דו"ח סטטוס), לא רשימה.
**AC:**
- [ ] שינוי שם ל-`/reports/annual-report-status` (או דומה)

---

## 🔶 פערי Filter ו-Search

### 50. חיפוש טקסטואלי — 4 שמות שונים
**בעיה:** `search`, `query`, `client_name`, `client_search` — לאותה פעולה.
**AC:**
- [ ] שם אחיד אחד (`search`) בכל ה-endpoints
- [ ] ה-frontend מעודכן

### 51. תאריכי טווח — 7 פטרנים
**בעיה:** `from`/`to`, `from_date`/`to_date`, `date_from`/`date_to`, `issued_after`/`issued_before`, `due_after`/`due_before`, `start_year`/`end_year`, `from_year`/`to_year`.
**AC:**
- [ ] פטרן אחיד אחד (`{field}_after`/`{field}_before`) בכל ה-endpoints
- [ ] ה-frontend מעודכן

### 52. 🔍 16 endpoints לרשימות ללא filters
**בעיה:** fetch עיוור.
**AC:**
- [ ] בירור לכל אחד מאלה שעלולים לגדול: `GET /binders/open`, `/signature-requests/pending`, `/clients/{id}/annual-reports`, `/vat/clients/{id}/work-items`, `/clients/{id}/businesses`, `/audit/{entity_type}/{entity_id}` — האם צריך filters?

---

## 🔴 Error Schemas

### 53. אין schema גנרי לשגיאות עסקיות
**בעיה:** רק `HTTPValidationError` (422) מוגדר. אין תיעוד ל-400/403/409/500.
**AC:**
- [ ] schema אחיד `ErrorResponse { code, message, details? }`
- [ ] משויך ל-responses השגיאה בכל ה-endpoints
- [ ] ה-frontend מסתמך על מבנה ידוע

### 54. `GET /annual-reports/{id}/charges` — response schema ריק
**בעיה:** מחזיר `{}` במקום schema מוגדר.
**AC:**
- [ ] schema מפורש לתגובה

---

## 🔍 ממצאים נוספים

### 55. שני audit endpoints עם shapes שונות
**בעיה:** `VatAuditTrailResponse` = `{items, total}`; `EntityAuditTrailResponse` = `{items, total, limit, offset}`.
**AC:**
- [ ] envelope אחיד לשני ה-audit endpoints

### 56. 🔍 VAT CREATE רק global, לא per-client
**בעיה:** בניגוד לדומיינים אחרים שיש להם `POST /clients/{id}/X`.
**AC:**
- [ ] בירור: להוסיף `POST /vat/clients/{id}/work-items` לעקביות?

### 57. inline enum ב-correspondence `order`
**בעיה:** `['asc','desc']` inline במקום `$ref`.
**AC:**
- [ ] enum מוגדר כ-schema נפרד ומופנה אליו

### 58. 🔍 `BinderHandoverToClientRequest` — body אופציונלי
**בעיה:** `anyOf: [schema, null]` — POST ללא body מותר.
**AC:**
- [ ] בירור: מכוון? אם כן — לתעד. אם לא — להסיר את ה-null

---

## 🔺 חוסר עקביות בטיפוסים (Type Conflicts)

### 59. 🔍 `created_at` — `date-time` מול `date`
**בעיה:** כמעט כולם `date-time`, אבל `VatInvoiceResponse` הוא `date`.
**AC:** בירור: חשבונית VAT צריכה שעה? להחליט ולאחד.

### 60. 🔍 `filing_deadline` — 3 טיפוסים
**בעיה:** `string` ללא format / `date-time` / `date` — לאותו שדה לוגי.
**AC:** בירור: מהו הטיפוס הנכון? לאחד בכל 4 ה-schemas.

### 61. `occurred_at` — אותו שדה, שני טיפוסים (כנראה באג)
**בעיה:** `CorrespondenceCreateRequest`=`date-time`, `CorrespondenceUpdateRequest`=`string` גולמי.
**AC:** לתקן את Update ל-`date-time`.

### 62. 🔍 `amount` — `decimal` מול `string` גולמי
**בעיה:** רוב המקומות `decimal`, `AttentionBoardItem` גולמי.
**AC:** בירור: האם זה סכום כספי? אם כן — `decimal`.

### 63. שדות פיננסיים — `string` גולמי ב-Update (כנראה באג)
**בעיה:** `gross_amount`, `paid_amount`, `expected_amount`, `recognition_rate`, `other_credits` הם `decimal` ב-Create/Response אבל `string` גולמי ב-Update.
**AC:** לתקן את Update ל-`decimal` בכל אלה.

### 64. 🔍 `id` — `integer` מול `string`
**בעיה:** רוב הישויות `integer`. `WorkQueueItem.id` הוא `string` — **מאושר כמכוון** (יש לו regex `^\w+:\d+$`, composite ID כמו `vat_work_item:42`). נשאר רק `AttentionBoardItem.id` כ-`string` ללא הסבר.
**AC:** בירור: האם `AttentionBoardItem.id` גם composite? אם כן — להוסיף regex ולתעד. אם לא — `integer`.

### 65. 🔍 `counterparty_id` — `string` במקום `integer`
**בעיה:** ב-VatInvoice schemas.
**AC:** בירור: מספר עוסק/ח.פ (string מכוון) או FK פנימי?

### 66. 🔍 שיעורים כ-`number` במקום `decimal`
**בעיה:** `collection_rate`, `completion_rate`, `compliance_rate`, `effective_rate`, `credit_points`, `rate` כ-float.
**AC:** בירור: float מקובל לשיעורים, או `decimal` כמו שאר הכספים (סיכון עיגול)?

---

## 🔻 בעיות עיצוב Schema ו-Enum

### 67. 🔍 `ExpenseCategory` vs `ExpenseCategoryType`
**בעיה:** שני enums לסיווג הוצאות, 7 ערכים משותפים + ערכים שונים.
**AC:** בירור: מה ההבדל ואיפה כל אחד? לאחד או לתעד.

### 68. `ReminderActionType` — UPPER_CASE חריג
**בעיה:** היחיד ב-UPPER_CASE מתוך 52 enums (השאר snake_case).
**AC:** המרה ל-`snake_case` (`create_task` וכו') + עדכון frontend.

### 69. 🔍 `NotificationPageSize` — enum `[25, 50]`
**בעיה:** pagination מוגבל ל-2 ערכים דרך enum.
**AC:** בירור: ההגבלה מכוונת? אם לא — param רגיל עם min/max.

---

## 🐞 באגים מובהקים (תיקון ישיר, ללא בירור)

### 73. שדות `required` שהם גם `nullable` — 18 שדות (סתירה לוגית)
**בעיה:** שדה לא יכול להיות גם חובה וגם null. זה מבלבל codegen — ב-TypeScript ייווצר `field: string | null` אך מסומן חובה. כנראה נובע מ-Pydantic models עם `Optional` ללא default.
**מושפעים:** `VatPeriodRow` (8 שדות: `filed_at`, `period_type`, `final_vat_amount`, `is_overdue`, `submission_deadline`, `statutory_deadline`, `extended_deadline`, `days_until_deadline`), `EntityAuditLogResponse` + `VatAuditLogResponse` (`old_value`, `new_value`, `note`), `TaxCalculationSaveResponse` (`refund_due`, `tax_due`), `TaxCalendarSummaryResponse` (`start_year`, `end_year`).
**AC:**
- [ ] לכל שדה: להחליט required (לא-null) או optional (ניתן להיעדר) — ולתקן את ה-Pydantic model בהתאם
- [ ] codegen מייצר טיפוסים עקביים (אין `required` + `| null` יחד)

### 74. `page_size` max לא עקבי — 100 מול 200
**בעיה:** 25 endpoints מגבילים ל-100, 16 ל-200, ו-`GET /notifications` ללא max כלל. אין הגיון בחלוקה — `GET /annual-reports`=200 אבל `GET /clients/{id}/annual-reports`=100, לאותו סוג ישות.
**AC:**
- [ ] מקסימום אחיד אחד לכל ה-endpoints (להחליט 100 או 200)
- [ ] `GET /notifications` מקבל max מוגדר
- [ ] בדיקה: בקשה מעל המקסימום מוחזרת עם `422`

### 75. `period` — regex validation רק ב-1 מתוך 19 schemas
**בעיה:** השדה `period` (פורמט `YYYY-MM`) מופיע ב-19 schemas, אבל ה-regex `^\d{4}-(0[1-9]|1[0-2])$` קיים רק ב-`AdvancePaymentCreateRequest`. שאר ה-input schemas (`VatWorkItemCreateRequest`, `ChargeCreateRequest`) מקבלים `period` חופשי — אפשר לשלוח `period: "garbage"` ל-VAT work item והוא יתקבל.
**AC:**
- [ ] ה-regex מוחל על כל ה-Create/input schemas שמקבלים `period`
- [ ] עדיף: type משותף (`PeriodStr`) שמרכז את ה-validation במקום אחד
- [ ] בדיקה: `period` לא חוקי מוחזר עם `422`

### 76. `POST /annual-reports/{id}/expenses` ו-`/income` — אין GET-list להורה
**בעיה:** לשתי הקולקציות יש רק POST — אין GET שמחזיר את שורות ההכנסה/הוצאה. השורות ניתנות ל-PATCH/DELETE, אבל אי אפשר לשלוף את הרשימה בנפרד; היא מגיעה רק מקוננת ב-`AnnualReportDetailResponse` השמן (52 שדות). חוסר עקביות בולט: `annex/{schedule}` כן יש לו GET.
**AC:**
- [ ] `GET /annual-reports/{id}/expenses` — מחזיר שורות הוצאה עם envelope אחיד
- [ ] `GET /annual-reports/{id}/income` — מחזיר שורות הכנסה
- [ ] מאפשר רענון השורות בלי טעינת הדוח המלא

### 77. `binders/{id}/intakes/{intake_id}` — יש PATCH אבל אין DELETE
**בעיה:** אפשר לעדכן intake בודד אבל לא למחוק. אם אפשר להוסיף (receive) ולעדכן — צריך גם להסיר.
**AC:**
- [ ] `DELETE /binders/{binder_id}/intakes/{intake_id}` — מחיקת intake בודד
- [ ] מחזיר `204` + `404` אם לא קיים

---

## ⚠️ תיקוני עקביות חוצי-מערכת

### 70. `client_id` vs `client_record_id`
**בעיה:** שני path params מעורבבים ב-spec.
**AC:** איחוד ל-`client_record_id` (ה-anchor התפעולי) בכל ה-paths + frontend.

### 71. `history` vs `audit`
**סטטוס:** בוצע. `audit` הוא הדפוס הקנוני לנתיבי audit trail; נתיבי binders/annual-reports הועברו ל-`/audit`.
**AC:** בחירת דפוס אחד + יישום בכל הדומיינים.

### 72. `restore` לא עקבי
**בעיה:** קיים ב-clients/businesses, חסר ב-binders/charges/documents.
**AC:** restore בכל דומיין עם soft delete (מקושר לפריטים 22, 30).
