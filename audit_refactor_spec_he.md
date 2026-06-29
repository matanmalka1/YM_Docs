**אפיון ריפקטור Audit Logs**

**YM Tax CRM — מצב רצוי סופי**

# **1. תקציר החלטה**

**המצב הרצוי הסופי הוא שני מודלי Audit בלבד:**

- EntityAuditLog — כל שינוי עסקי במערכת: לקוחות, עסקים, מע״מ, מקדמות,
  דוחות שנתיים, מסמכים, חיובים, קלסרים, חתימות, משימות, הערות ועוד.

- UserAuditLog — אירועי auth/security בלבד: התחברות, כשלון התחברות,
  logout, איפוס סיסמה, חסימת גישה, הפעלה/השבתה של משתמשים וכדומה.

לא יהיה מודל Audit נפרד לכל דומיין. המודלים הקיימים כמו VatAuditLog,
AnnualReportStatusHistory, BinderLifecycleLog, BinderIntakeEditLog
ו-SignatureAuditEvent מתמזגים לתוך EntityAuditLog.

# **2. העיקרון המנחה**

Audit צריך לענות על שאלה פשוטה: מי שינה מה, מתי, למה ובאיזה הקשר.

| **שדה**               | **משמעות**  | **מה נכנס לשם**                                                                                       |
|-----------------------|-------------|-------------------------------------------------------------------------------------------------------|
| old_value / new_value | השינוי עצמו | השדות שהשתנו והערכים לפני/אחרי. למשל amount, status, due_date.                                        |
| metadata_json         | הקשר        | client_record_id, business_id, tax_year, period, invoice_number, source screen, ip/user_agent כשצריך. |
| note                  | סיבה אנושית | הסבר שהמשתמש הכניס או סיבה עסקית: “תיקון לפי חשבונית מקורית”.                                         |

**כלל חד: אם זה השדה שהשתנה — old_value/new_value. אם זה עוזר להבין
למי/איפה/מאיפה השינוי — metadata_json. אם זה הסבר חופשי — note.**

# **3. EntityAuditLog — המודל המרכזי**

EntityAuditLog מכסה את כל השינויים העסקיים במערכת. הוא כולל גם שינויים
רגילים, גם שינויי סטטוס, וגם אירועים עסקיים כמו חתימה, הגשה, תשלום,
מסירה, אישור/דחייה של מסמך ועוד.

> class EntityAuditLog(Base):  
> \_\_tablename\_\_ = "entity_audit_logs"  
>   
> id: int  
>   
> entity_type: str \# client / vat_invoice / advance_payment /
> annual_report / binder / signature_request  
> entity_id: int \# id של הרשומה שהשתנתה  
>   
> action: str \# פעולה ברורה: vat_invoice.amount_changed /
> annual_report.status_changed  
>   
> performed_by: int \| None \# user id, nullable עבור system/external
> signer  
> actor_type: str \# user / system / external_signer  
>   
> old_value: dict \| None \# JSONB — מה היה לפני  
> new_value: dict \| None \# JSONB — מה אחרי  
> metadata_json: dict \| None# JSONB — הקשר נוסף  
> note: str \| None \# למה / הסבר אנושי  
>   
> performed_at: datetime

## **3.1 אינדקסים מומלצים**

> Index("idx_entity_audit_entity_time", "entity_type", "entity_id",
> "performed_at")  
> Index("idx_entity_audit_action_time", "action", "performed_at")  
> Index("idx_entity_audit_actor_time", "performed_by", "performed_at")  
> Index("idx_entity_audit_performed_at", "performed_at")

ב-Postgres יש להשתמש ב-JSONB עבור old_value, new_value ו-metadata_json.
לא Text. Text יוצר parsing ידני ומונע שאילתות יעילות על payload.

# **4. UserAuditLog — רק Auth/Security**

UserAuditLog נשאר נפרד כי אירועי auth/security אינם בהכרח שינוי על
entity עסקי. לדוגמה: login_failure עם email שלא קיים במערכת. אין
entity_id תקף.

> class UserAuditLog(Base):  
> \_\_tablename\_\_ = "user_audit_logs"  
>   
> id: int  
>   
> action: str \# login_success / login_failure / logout /
> password_reset  
> status: str \# success / failure  
>   
> actor_user_id: int \| None  
> target_user_id: int \| None  
>   
> email: str \| None \# קריטי בכשלון התחברות  
> reason: str \| None  
> metadata_json: dict \| None \# ip, user_agent, request_id, etc.  
>   
> created_at: datetime

שינוי user entity קיים, כמו שינוי role או השבתת משתמש על ידי admin, יכול
להירשם גם ב-EntityAuditLog עם entity_type="user". אבל אירועי התחברות,
session, password reset וכשלונות אבטחה נשארים ב-UserAuditLog.

# **5. Action naming**

אסור להשתמש בפעולות כלליות כמו updated/changed/edit בלבד. action חייב
להיות ספציפי וקריא.

| **לא טוב**     | **טוב**                                   |
|----------------|-------------------------------------------|
| updated        | client.updated                            |
| changed        | vat_invoice.amount_changed                |
| status_changed | annual_report.status_changed              |
| signed         | signature_request.signed                  |
| paid           | advance_payment.marked_paid / charge.paid |

המלצה: להחזיק constants בלבד, למשל ACTION_VAT_INVOICE_AMOUNT_CHANGED. לא
לכתוב raw strings בתוך services.

# **6. דוגמאות כתיבה**

## **6.1 שינוי סכום חשבונית מע״מ**

> EntityAuditLog(  
> entity_type="vat_invoice",  
> entity_id=invoice.id,  
> action="vat_invoice.amount_changed",  
> performed_by=user.id,  
> actor_type="user",  
> old_value={"net_amount": "10000.00", "vat_amount": "1700.00"},  
> new_value={"net_amount": "12000.00", "vat_amount": "2040.00"},  
> metadata_json={  
> "client_record_id": client.id,  
> "vat_work_item_id": work_item.id,  
> "invoice_number": invoice.invoice_number,  
> "tax_year": 2026,  
> "period": "2026-05",  
> "source": "vat_invoice_edit_modal",  
> },  
> note="תיקון סכום לפי חשבונית מקורית",  
> )

## **6.2 שינוי סכום מקדמה**

> EntityAuditLog(  
> entity_type="advance_payment",  
> entity_id=advance_payment.id,  
> action="advance_payment.amount_changed",  
> performed_by=user.id,  
> actor_type="user",  
> old_value={"expected_amount": "1500.00"},  
> new_value={"expected_amount": "1800.00"},  
> metadata_json={  
> "client_record_id": advance_payment.client_record_id,  
> "tax_year": 2026,  
> "period": "2026-05",  
> "source": "advance_payment_details_page",  
> },  
> note="עודכן לפי דרישת מס חדשה",  
> )

## **6.3 חתימה דיגיטלית**

> EntityAuditLog(  
> entity_type="signature_request",  
> entity_id=signature_request.id,  
> action="signature_request.signed",  
> performed_by=None,  
> actor_type="external_signer",  
> old_value={"status": "pending_signature"},  
> new_value={"status": "signed"},  
> metadata_json={  
> "client_record_id": signature_request.client_record_id,  
> "signer_name": signature_request.signer_name,  
> "signer_email": signature_request.signer_email,  
> "ip_address": signer_ip,  
> "user_agent": signer_user_agent,  
> "content_hash": signature_request.content_hash,  
> "signed_document_key": signature_request.signed_document_key,  
> },  
> note=None,  
> )

# **7. מה מתמזג / נמחק**

| **מודל קיים**             | **מצב סופי**          | **הערה**                                                                                |
|---------------------------|-----------------------|-----------------------------------------------------------------------------------------|
| EntityAuditLog            | נשאר ומשתדרג          | JSONB, metadata_json, actor_type, performed_by nullable.                                |
| UserAuditLog              | נשאר ומשתדרג          | רק auth/security. metadata_json כ-JSONB.                                                |
| VatAuditLog               | נמחק → EntityAuditLog | work_item_id הופך ל-entity_id עם entity_type="vat_work_item" או vat_invoice לפי הפעולה. |
| AnnualReportStatusHistory | נמחק → EntityAuditLog | status changes נרשמים כ-action="annual_report.status_changed".                          |
| BinderLifecycleLog        | נמחק → EntityAuditLog | location/capacity status changes נכנסים כפעולות binder.\*.                              |
| BinderIntakeEditLog       | נמחק → EntityAuditLog | עריכות intake/material נכנסות כ-entity_type מתאים.                                      |
| SignatureAuditEvent       | נמחק → EntityAuditLog | signature_request.\* כולל metadata forensic.                                            |

# **8. כיסוי Audit לפי טבלאות**

הרשימה הבאה מגדירה אילו טבלאות צריכות לקבל EntityAuditLog ואילו לא. זה
מבוסס על המיגרציה הראשונית ועל משמעות עסקית של כל טבלה.

| **טבלה**                  | **Audit?** | **מה חשוב לתעד**                                                 |
|---------------------------|------------|------------------------------------------------------------------|
| legal_entities            | חובה       | מספרי זהות/ח.פ, שם רשמי, פרופיל מס, תדירות דיווח, מקדמות, מחזור. |
| persons                   | חובה       | פרטי אדם, תעודה, טלפון, מייל, כתובת.                             |
| person_legal_entity_links | חובה       | בעלים/מורשה חתימה/בעל שליטה.                                     |
| client_records            | חובה       | סטטוס לקוח, שיוך רו״ח, notes, soft delete/restore.               |
| businesses                | חובה       | שם עסק, סטטוס, תאריכי פתיחה/סגירה, פרטי קשר.                     |
| authority_contacts        | חובה       | אנשי קשר מול רשויות.                                             |
| entity_notes              | חובה       | יצירה/עריכה/מחיקה של הערות.                                      |

| **טבלה**         | **Audit?** | **מה חשוב לתעד**                                                                              |
|------------------|------------|-----------------------------------------------------------------------------------------------|
| advance_payments | חובה       | expected/paid/override/calculated amount, status, paid_at, due_date override, payment_method. |
| charges          | חובה       | amount, status, issued/paid/canceled, cancellation_reason.                                    |
| invoices         | חובה       | חשבונית חיצונית שמחוברת לחיוב: provider, external_invoice_id, document_url.                   |
| vat_work_items   | חובה       | status, assignment, final_vat_amount, override, filed/submission fields, due date override.   |
| vat_invoices     | חובה       | net/vat amounts, invoice_number/date, counterparty, deduction_rate, category, exceptional.    |

| **טבלה**                    | **Audit?** | **מה חשוב לתעד**                                                                           |
|-----------------------------|------------|--------------------------------------------------------------------------------------------|
| annual_reports              | חובה       | status, assigned_to, deadline, submitted_at, references, assessment/refund/tax_due, notes. |
| annual_report_details       | חובה       | credits, donation, pension, amendment_reason, client approval.                             |
| annual_report_income_lines  | חובה       | amount, source_type, description.                                                          |
| annual_report_expense_lines | חובה       | amount, category, recognition_rate, document reference.                                    |
| annual_report_credit_points | חובה       | reason, points, notes.                                                                     |
| annual_report_schedules     | חובה       | required/complete, completed_by, completed_at.                                             |
| annual_report_annex_data    | חובה       | data JSON של נספחים, line_number, notes.                                                   |

| **טבלה**                | **Audit?** | **מה חשוב לתעד**                                                                      |
|-------------------------|------------|---------------------------------------------------------------------------------------|
| binders                 | חובה       | location_status, capacity_status, handover fields, notes, delete.                     |
| binder_intakes          | חובה       | received_at, received_by, notes.                                                      |
| binder_intake_materials | חובה       | material_type, period, business/report/vat links, description.                        |
| binder_handovers        | חובה       | מסירה ללקוח, recipient, period until, notes.                                          |
| binder_handover_binders | חובה       | שיוך קלסר למסירה.                                                                     |
| permanent_documents     | חובה       | upload/replace/approve/reject/delete, status, version, links, storage metadata.       |
| signature_requests      | חובה       | created/sent/viewed/signed/declined/canceled/expired + forensic metadata.             |
| tasks                   | חובה       | assignment, status, priority, due date, complete/cancel/delete.                       |
| correspondence_entries  | חובה       | יומן תקשורת מול לקוח/רשות: create/update/delete.                                      |
| notifications           | חלקי       | log על created/sent/failed/skipped כשמשתמש יזם או כשזה עסקית חשוב. לא כל retry פנימי. |
| reminders               | חלקי       | created/canceled/fired/failed. עדיפות נמוכה.                                          |

| **טבלה**                   | **Audit?** | **החלטה**                                                                                     |
|----------------------------|------------|-----------------------------------------------------------------------------------------------|
| deadline_rules             | תלוי       | אם יש UI לעריכה — audit. אם static seed — לא דחוף.                                            |
| tax_calendar_entries       | חלקי       | לא לוג לכל entry שנוצר bulk. כן log אחד tax_calendar.generated. עריכה ידנית של deadline — כן. |
| idempotency_keys           | לא         | תשתיתי. לא EntityAuditLog.                                                                    |
| password_reset_tokens      | לא         | ה-token עצמו לא audit. אירועי reset הולכים ל-UserAuditLog.                                    |
| users                      | תלוי       | שינוי user קיים יכול להיכנס ל-EntityAuditLog; auth/security ל-UserAuditLog.                   |
| reminders internal retries | לא/חלקי    | כשלון משמעותי כן, רעש retry טכני לא.                                                          |

# **9. Entity types מומלצים**

> ENTITY_LEGAL_ENTITY = "legal_entity"  
> ENTITY_PERSON = "person"  
> ENTITY_PERSON_LEGAL_ENTITY_LINK = "person_legal_entity_link"  
> ENTITY_CLIENT = "client"  
> ENTITY_BUSINESS = "business"  
> ENTITY_AUTHORITY_CONTACT = "authority_contact"  
> ENTITY_NOTE = "note"  
> ENTITY_ADVANCE_PAYMENT = "advance_payment"  
> ENTITY_CHARGE = "charge"  
> ENTITY_INVOICE = "invoice"  
> ENTITY_VAT_WORK_ITEM = "vat_work_item"  
> ENTITY_VAT_INVOICE = "vat_invoice"  
> ENTITY_ANNUAL_REPORT = "annual_report"  
> ENTITY_ANNUAL_REPORT_DETAIL = "annual_report_detail"  
> ENTITY_ANNUAL_REPORT_INCOME_LINE = "annual_report_income_line"  
> ENTITY_ANNUAL_REPORT_EXPENSE_LINE = "annual_report_expense_line"  
> ENTITY_ANNUAL_REPORT_CREDIT_POINT = "annual_report_credit_point"  
> ENTITY_ANNUAL_REPORT_SCHEDULE = "annual_report_schedule"  
> ENTITY_ANNUAL_REPORT_ANNEX_DATA = "annual_report_annex_data"  
> ENTITY_BINDER = "binder"  
> ENTITY_BINDER_INTAKE = "binder_intake"  
> ENTITY_BINDER_INTAKE_MATERIAL = "binder_intake_material"  
> ENTITY_BINDER_HANDOVER = "binder_handover"  
> ENTITY_DOCUMENT = "document"  
> ENTITY_SIGNATURE_REQUEST = "signature_request"  
> ENTITY_TASK = "task"  
> ENTITY_CORRESPONDENCE = "correspondence"  
> ENTITY_NOTIFICATION = "notification"  
> ENTITY_REMINDER = "reminder"  
> ENTITY_TAX_CALENDAR = "tax_calendar"

# **10. Metadata policy**

metadata_json צריך להיות קונטקסט מינימלי שעוזר לבנות timeline, dashboard
ו-history בלי joins מיותרים. הוא לא מחליף את old_value/new_value.

| **סוג ישות**              | **metadata_json מומלץ**                                                                                           |
|---------------------------|-------------------------------------------------------------------------------------------------------------------|
| vat_invoice               | client_record_id, vat_work_item_id, business_id, invoice_number, tax_year, period, source.                        |
| advance_payment           | client_record_id, tax_year, period, annual_report_id, source.                                                     |
| annual_report child lines | annual_report_id, client_record_id, tax_year, line_number/schedule.                                               |
| document                  | client_record_id, business_id, annual_report_id, binder_id, document_type, version.                               |
| signature_request         | client_record_id, business_id, annual_report_id, document_id, signer_email, ip_address, user_agent, content_hash. |
| binder/binder_intake      | client_record_id, binder_number, period_start, period_end.                                                        |
| task                      | client_record_id, source_domain, source_id, assigned_to_user_id, assigned_role.                                   |

# **11. מה לא לעשות**

- לא להפוך metadata_json לפח של כל השדות. השינוי עצמו חייב להיות
  ב-old_value/new_value.

- לא להשתמש ב-Text עבור JSON. להשתמש ב-JSONB ב-Postgres.

- לא לכתוב action כללי מדי כמו updated לכל דבר משמעותי.

- לא לערבב login/security failures בתוך EntityAuditLog.

- לא ליצור log על כל retry טכני או generated row בכמויות. לרשום פעולה
  עסקית אחת ברמת batch.

- לא להסתמך על docstring ל-immutability. אם רוצים append-only אמיתי,
  צריך מדיניות DB או לפחות לא לחשוף update/delete ב-repository.

# **12. סדר יישום מומלץ**

| **שלב** | **תוכן**                                                                                                                         | **למה**                                                     |
|---------|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| 1       | שדרוג EntityAuditLog: JSONB, metadata_json, actor_type, performed_by nullable, indexes.                                          | בסיס לכל המיזוגים.                                          |
| 2       | שדרוג UserAuditLog: metadata_json JSONB, action/status strings או enums קיימים.                                                  | הפרדת security נשמרת.                                       |
| 3       | מיזוג VatAuditLog → EntityAuditLog.                                                                                              | הכי זול; invoice_id לא בשימוש כקריאה.                       |
| 4       | מיזוג AnnualReportStatusHistory → EntityAuditLog.                                                                                | status changes הופכים לפעולות annual_report.status_changed. |
| 5       | מיזוג BinderLifecycleLog + BinderIntakeEditLog → EntityAuditLog.                                                                 | מאחד lifecycle/intake edits.                                |
| 6       | מיזוג SignatureAuditEvent → EntityAuditLog.                                                                                      | signature lifecycle + forensic metadata.                    |
| 7       | הוספת audit חסר לפי עדיפות: legal_entities/persons, advance_payments, vat_invoices, documents, annual report child lines, tasks. | סוגר פערי accountability.                                   |
| 8       | עדכון timeline/dashboard/repositories/schemas/OpenAPI/tests.                                                                     | יישור צרכנים.                                               |

# **13. Acceptance Criteria**

- במערכת נשארים רק שני מודלי audit: EntityAuditLog ו-UserAuditLog.

- כל טבלאות ה-audit הישנות מוסרות מהמודלים, repos, schemas, seed
  והמיגרציה.

- EntityAuditLog תומך ב-JSONB עבור old_value, new_value, metadata_json.

- כל שינוי עסקי משמעותי מהרשימה בפרק 8 נכתב ל-EntityAuditLog.

- אירועי auth/security נכתבים ל-UserAuditLog בלבד.

- timeline/dashboard ממשיכים לעבוד בלי כפילויות ובלי \_DEDUP_ACTIONS
  מפוזר שלא לצורך.

- אין שימוש ב-raw action strings בשירותים; הכל דרך constants.

- בדיקות עוברות: audit, vat, binders, annual_reports,
  signature_requests, users, timeline/dashboard.

# **14. סיכום קצר**

**היעד הסופי הוא פשוט: EntityAuditLog הוא ההיסטוריה העסקית של המערכת.
UserAuditLog הוא היסטוריית אבטחה והתחברות. כל השאר הוא כפילות היסטורית
ומוסר בריפקטור.**
