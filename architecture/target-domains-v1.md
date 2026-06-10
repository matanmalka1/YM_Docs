# CRM Domains / Subdomains / Actions

---

# 1. Clients Domain

אחראי על הקשר התפעולי בין המשרד לבין הלקוח.
זהות + lifecycle + ownership בלבד.

## Client Management

- create_client()
- get_client()
- get_client_summary()
- get_client_operational_status()
- list_clients()
- update_client()
- soft_delete_client()
- restore_client()

## Client Lifecycle

- activate_client()
- deactivate_client()
- freeze_client()
- unfreeze_client()
- close_client()
- reopen_client()

## Client Ownership

- assign_accountant()
- change_accountant()
- unassign_accountant()

- assign_secretary()
- change_secretary()
- unassign_secretary()

- transfer_client_ownership()

## Client Tags

- add_client_tag()
- remove_client_tag()
- list_client_tags()

- bulk_add_client_tags()
- bulk_remove_client_tags()

---

# 2. Legal Entities Domain

הישות המשפטית היא מקור האמת לכל הסיווגים המיסויים.
סיווג מע"מ, תדירות דיווח, וסוג מס הכנסה — כולם שייכים לישות, לא לפעילות.
עוסק יחיד = סיווג אחד לכל עסקיו יחד (חוק מע"מ, סעיף 1).
חברה בע"מ = תמיד עוסק מורשה מהשקל הראשון (תקנה 13(7)).

## Legal Entity Management

- create_legal_entity()
- get_legal_entity()
- list_legal_entities()
- update_legal_entity()
- soft_delete_legal_entity()
- restore_legal_entity()

## Legal Entity Identification

- update_tax_id()
- update_vat_number()
- validate_legal_entity_identity()

## Legal Entity Address

- update_legal_entity_address()
- update_legal_entity_phone()
- update_legal_entity_email()

## Legal Entity Tax Profile

- get_legal_entity_tax_profile()

- update_vat_registration_type()
- update_vat_reporting_frequency()

- update_income_tax_type()

- update_national_insurance_type()

- update_tax_file_number()

## Legal Entity Status

- activate_legal_entity()
- deactivate_legal_entity()
- close_legal_entity()
- reopen_legal_entity()

---

# 3. Businesses Domain

Business מייצג פעילות עסקית בלבד.
לא מחזיק סיווגי מס — אלה שייכים ל-LegalEntity.
primary anchor תמיד client_record_id; העסק הוא תיוג, לא הבעלים.

## Business Management

- create_business()
- get_business()
- list_businesses()
- list_client_businesses()
- list_active_client_businesses()

- update_business()
- soft_delete_business()
- restore_business()

## Business Lifecycle

- open_business_activity()
- close_business_activity()
- reopen_business_activity()

## Business Assignment

- link_business_to_client()
- unlink_business_from_client()

- transfer_business_to_client()

- set_primary_business()

## Business Activity

- update_business_name()
- update_business_status()

## Business Contact Overrides

- update_business_phone()
- update_business_email()
- update_business_address()

---

# 4. Contacts Domain

אנשי קשר של הלקוח — לא רשויות.
Authority Contacts הם דומיין נפרד.

## Contact Management

- create_contact()
- get_contact()
- list_contacts()

- update_contact()
- delete_contact()

## Client Contacts

- add_contact_to_client()
- remove_contact_from_client()
- list_client_contacts()

## Contact Roles

- set_primary_contact()
- unset_primary_contact()

- assign_contact_role()
- remove_contact_role()

## Contact Communication

- add_contact_email()
- remove_contact_email()

- add_contact_phone()
- remove_contact_phone()

- update_preferred_contact_method()

---

# 5. Authority Contacts Domain

גורמי רשות ומוסדות — נפרד מאנשי קשר של הלקוח.
תמיד client-scoped בלבד, ללא business_id.

## Authority Contact Management

- create_authority_contact()
- get_authority_contact()
- list_authority_contacts()
- list_client_authority_contacts()
- update_authority_contact()
- soft_delete_authority_contact()
- restore_authority_contact()

## Authority Contact Assignment

- link_authority_contact_to_client()
- unlink_authority_contact_from_client()

---

# 6. Notes Domain

EntityNote גנרי עם entity_type + entity_id.
כל note חייב client_record_id לצורכי permission/query.
משרת כל דומיין — לא שייך ל-Clients בלבד.

## Note Management

- create_note()
- get_note()
- list_notes()
- update_note()
- delete_note()

## Scoped Notes

- list_client_notes()
- list_business_notes()
- list_entity_notes()

## Note Actions

- pin_note()
- unpin_note()

---

# 7. Documents Domain

מסמכים מתויגים ל-client_record_id (חובה) ואופציונלית ל-business_id.
scope=BUSINESS מחייב business_id; scope=CLIENT מחייב business_id=NULL.
ישויות ראשיות: soft delete. שורות פירוט (annexes, lines): hard delete.

## Document Management

- upload_document()
- get_document()
- list_documents()
- update_document_metadata()
- soft_delete_document()

## Client Documents

- upload_client_document()
- list_client_documents()
- list_business_documents()

## Permanent Documents

- create_permanent_document()
- get_permanent_document()
- list_client_permanent_documents()
- list_expiring_permanent_documents()
- list_expired_permanent_documents()
- update_permanent_document()
- mark_permanent_document_valid()
- mark_permanent_document_expired()
- replace_permanent_document()

## Required Documents

- create_required_document()
- list_required_documents()
- list_missing_documents()
- mark_document_request_sent()
- mark_document_received()
- cancel_required_document()
- expire_required_document()
- reopen_required_document()

## Document Organization

- create_document_folder()
- rename_document_folder()
- soft_delete_document_folder()
- move_folder_documents()
- move_document()

## Document Classification

- classify_document()
- tag_document()
- untag_document()

## Document Access

- get_document_download_url()
- get_document_preview_url()

---

# 8. Signature Requests Domain

## Signature Request Management

- create_signature_request()
- create_signature_request_for_document()
- get_signature_request()
- get_signature_request_status()
- list_signature_requests()
- list_client_signature_requests()
- list_pending_signature_requests()
- list_expired_signature_requests()
- cancel_signature_request()

## Signature Request Lifecycle

- approve_signature_request()
- decline_signature_request()
- resend_signature_request()
- expire_signature_request()

## Signing

- get_signing_url()
- verify_signing_token()

---

# 9. Tasks Domain

## Task Management

- create_task()
- get_task()
- list_tasks()
- update_task()
- delete_task()

## Client Tasks

- create_client_task()
- list_client_tasks()

## Task Assignment

- assign_task()
- reassign_task()
- unassign_task()

## Task Lifecycle

- complete_task()
- reopen_task()
- cancel_task()
- snooze_task()
- mark_task_overdue()

## Task Queries

- list_overdue_tasks()
- list_due_today_tasks()

## Task Priority

- set_task_priority()
- update_task_due_date()

---

# 10. Reminders Domain

Reminder = התראה בזמן. שונה מ-Task שהוא עבודה לביצוע.

## Reminder Management

- create_reminder()
- get_reminder()
- list_reminders()
- list_client_reminders()
- update_reminder()
- delete_reminder()

## Reminder Lifecycle

- trigger_reminder()
- snooze_reminder()
- dismiss_reminder()
- complete_reminder()

---

# 11. Alerts Domain

Alert = signal פנימי שנוצר מחוקים עסקיים בתוך המערכת.
שונה מ-Notification שהיא הודעה שנשלחת החוצה/למשתמש.

## Alert Management

- get_client_alerts()
- get_client_alert_summary()
- get_sidebar_alert_counts()
- mark_alert_read()
- mark_all_alerts_read()
- dismiss_alert()

## Alert Calculation

- calculate_client_alerts()
- refresh_client_alerts()
- suppress_alert()
- unsuppress_alert()

## Sidebar Alerts

- get_client_sidebar_alerts()
- get_clients_alerts_batch()

---

# 12. Charges Domain

Charge = source of truth לתשלום.
Invoice = מסמך חשבונאי בלבד — לא מחזיק סטטוס תשלום.

חיובים מתויגים ל-client_record_id (חובה) ואופציונלית ל-business_id.

## Charge Management

- create_charge()
- get_charge()
- list_charges()
- update_charge()
- soft_delete_charge()

## Client Charges

- list_client_charges()
- list_business_charges()
- get_client_charge_summary()

## Charge Lifecycle

- mark_charge_open()
- mark_charge_paid()
- mark_charge_overdue()
- cancel_charge()

## Payments

- record_payment()
- cancel_payment()
- reconcile_payment()

## Collections

- create_collection_task()
- mark_collection_started()
- mark_collection_completed()

---

# 13. Invoices Domain

Invoice = מסמך חשבונאי בלבד.
Charge הוא source of truth לתשלום — לא Invoice.
אם ספק חשבוניות מחזיר payment events → record_payment(charge_id, source="invoice_provider").

## Invoice Management

- create_invoice()
- create_invoice_for_charge()
- get_invoice()
- get_charge_invoice()
- list_invoices()
- list_client_invoices()
- update_invoice()

## Invoice Lifecycle

- issue_invoice()
- cancel_invoice()
- void_invoice()
- mark_invoice_sent()

## Invoice Access

- get_invoice_pdf()

---

# 14. Advance Payments Domain

קיים ב-Work Queue. soft delete על AdvancePayment.

## Advance Payment Management

- create_advance_payment()
- get_advance_payment()
- list_advance_payments()
- list_client_advance_payments()
- list_advance_payments_by_year()
- list_due_advance_payments()
- update_advance_payment()
- soft_delete_advance_payment()

## Advance Payment Lifecycle

- mark_advance_payment_open()
- mark_advance_payment_paid()
- mark_advance_payment_overdue()
- cancel_advance_payment()

## Advance Payment Summary

- get_client_advance_payment_summary()

---

# 15. VAT Domain

VAT work items שייכים ל-client_record_id בלבד — אין business_id.
tax/reporting workflows legally scoped to the client.

תהליך: data_entry_in_progress → ready_for_review → filed
ready_for_review: secretary + advisor
send_back + file: advisor only

## VAT Work Items

- create_vat_work_item()
- get_vat_work_item()
- list_vat_work_items()
- list_client_vat_work_items()
- update_vat_work_item()
- soft_delete_vat_work_item()

## VAT Lifecycle

- mark_material_missing()
- mark_material_received()
- mark_vat_in_progress()
- validate_vat_readiness()
- mark_ready_for_review()
- send_back_to_data_entry()
- mark_filed()
- mark_paid()
- reopen_vat_work_item()

## VAT Summaries

- get_client_vat_summary()
- get_open_vat_periods()
- get_missing_vat_materials()

---

# 16. Annual Reports Domain

תהליך: not_started → collecting_docs → in_preparation → ready_for_accountant_review → pending_client → submitted → closed
canceled: מצב סופי נפרד.
מעבר ל-submitted: advisor only + readiness checks.

## Annual Report Management

- create_annual_report()
- get_annual_report()
- list_annual_reports()
- list_client_annual_reports()
- update_annual_report()
- soft_delete_annual_report()

## Annual Report Lifecycle

- mark_collecting_docs()
- mark_in_preparation()
- validate_annual_report_readiness()
- mark_ready_for_accountant_review()
- send_back_to_preparation()
- mark_pending_client()
- mark_submitted()
- mark_closed()
- cancel_annual_report()
- reopen_annual_report()

## Financial Data

- update_income_lines()
- update_expense_lines()
- calculate_tax()
- calculate_final_balance()

## Annual Report Summary

- get_client_annual_reports_summary()
- get_annual_reports_status_summary()

---

# 17. National Insurance Domain

Target domain — קיים כ-target בלבד, טרם הופרד לתיקייה עצמאית בקוד.

## NI Records

- create_ni_record()
- get_ni_record()
- list_ni_records()
- list_client_ni_records()
- update_ni_record()

## NI Lifecycle

- mark_ni_open()
- mark_ni_reported()
- mark_ni_paid()
- mark_ni_overdue()

## NI Summary

- get_client_ni_summary()
- get_ni_balance_summary()

---

# 18. Tax Calendar Domain

DeadlineRule + TaxCalendarEntry עם due_date_original / due_date_effective.
התאמת תאריך לימי עסקים מנוהלת דרך due_date_effective.

## Tax Calendar Events

- create_tax_calendar_event()
- get_tax_calendar_event()
- list_tax_calendar_events()
- update_tax_calendar_event()
- delete_tax_calendar_event()

## Deadline Generation

- generate_tax_due_dates()
- generate_annual_schedule()

## Tax Calendar Queries

- get_upcoming_tax_deadlines()
- get_client_tax_calendar()
- get_overdue_tax_deadlines()

---

# 19. Binders Domain

## Binder Management

- create_binder()
- get_binder()
- list_binders()
- list_client_binders()
- list_binders_ready_for_handover()
- update_binder()
- soft_delete_binder()

## Binder Lifecycle

- receive_material()
- mark_binder_full()
- reopen_binder_capacity()
- mark_ready_for_handover()
- revert_ready_for_handover()
- handover_to_client()
- return_binder_to_office()

## Binder Status

- update_location_status()
- update_capacity_status()
- get_binder_status_summary()

---

# 20. Communication Domain

התכתבויות מתויגות ל-client_record_id (חובה) ואופציונלית ל-business_id כ-context.

## Email

- send_email()
- create_email_draft()
- resend_email()
- log_inbound_email()
- get_email_delivery_status()

## SMS

- send_sms()
- log_inbound_sms()
- get_sms_delivery_status()

## WhatsApp

- send_whatsapp()
- log_inbound_whatsapp()
- get_whatsapp_delivery_status()

## Calls

- log_phone_call()
- log_call_summary()

## Meetings

- create_meeting_note()
- update_meeting_note()

## Communication History

- list_client_communications()
- get_communication_thread()

---

# 21. Notifications Domain

Notification = הודעה שנשלחת החוצה/למשתמש.
שונה מ-Alert שהוא signal פנימי בתוך המערכת.

## Templates

- create_notification_template()
- update_notification_template()
- delete_notification_template()
- list_notification_templates()

## Sending

- preview_notification()
- resolve_notification_recipients()
- send_notification()
- send_bulk_notifications()
- resend_failed_notification()

## Policy

- check_notification_policy()
- block_notification()
- unblock_notification()

## Audit

- get_notification_status()
- list_notification_events()

---

# 22. Search Domain

כל החיפוש מרוכז כאן בלבד.
אין search_* בדומיינים האחרים.
כל list_* תומך pagination, filters, sort.

## Global Search

- global_search()

## Entity Search

- search_clients()
- search_legal_entities()
- search_businesses()
- search_contacts()
- search_documents()
- search_charges()
- search_tasks()

## Search Configuration

- list_available_filters()
- validate_search_filters()

---

# 23. Dashboard Domain

Read Model בלבד.
צורך נתונים מדומיינים אחרים — לא מחזיק לוגיקה עסקית.

## Dashboard

- get_dashboard_summary()
- get_client_dashboard()
- get_advisor_dashboard()
- get_secretary_dashboard()

---

# 24. Work Queue Domain

Application read model שמחשב priority rules.
לא משנה state של דומיינים.
לא מחזיק טבלה משלו — בונה WorkQueueItem סינתטי בכל request.
אוסף: VAT, annual reports, advance payments, charges, binders, tasks.

## Work Queue

- get_work_queue()
- get_priority_actions()
- get_overdue_items()
- get_upcoming_items()

## Work Queue Filters

- filter_work_queue_by_advisor()
- filter_work_queue_by_client()
- filter_work_queue_by_type()

---

# 25. Actions Domain

Application Layer — לא business domain קלאסי.
מרכז command definitions ו-execution metadata לפי דומיין.

## Action Registry

- list_available_actions()
- get_action_definition()

## Action Execution

- preview_action()
- execute_action()

## Domain Actions

- get_binder_actions()
- get_business_actions()
- get_charge_actions()
- get_annual_report_actions()
- get_vat_work_item_actions()

---

# 26. Reports Domain

BI תפעולי פעיל.

## Operational Reports

- generate_aging_report()
- generate_vat_compliance_report()
- generate_annual_report_status_report()
- generate_advance_payment_report()

## Report Access

- get_report_status()
- download_report()

## Dataset Exports

- export_excel()
- export_pdf()
- export_clients()
- export_documents()
- export_charges()
- export_compliance_report()

---

# 27. Audit Domain

## Audit Events

- create_audit_event()
- get_audit_event()
- list_audit_events()
- list_audit_events_by_actor()
- list_audit_events_by_entity()

## Client Audit

- get_client_audit_log()

## Entity Audit

- get_entity_audit_log()
- get_business_audit_log()

## User Activity

- get_user_activity_log()

## Change History

- get_change_history()

## Timeline

אין model/table עצמאי — aggregation layer בלבד.
מושך מ: client_records, permanent_documents, signature_requests,
audit events, binders, charges, notifications, tax.

- get_client_timeline()
- get_entity_timeline()

---

# 28. Users Domain

ניהול משתמשי המשרד. נפרד מ-Permissions.
Auth/session (login, refresh_token) — מחוץ למסמך זה, שייך ל-auth/core layer.

## User Management

- create_user()
- get_user()
- list_users()
- update_user()
- deactivate_user()
- reactivate_user()

## User Onboarding

- invite_user()
- reset_user_password()
- update_user_profile()

## Role Assignment

- assign_role()
- remove_role()

---

# 29. Permissions Domain

הרשאות לפי role בלבד (ADVISOR, SECRETARY).
ישויות כמו Document, Task, Charge, VAT יורשות הרשאה מהלקוח.
אין ACL פרטני ברמת ישות.

## Access Control

- check_client_access()
- check_role_permission()

## Role Queries

- list_user_permissions()

---

# 30. Client Import Domain

ייבוא לקוחות בלבד. Export נמצא ב-Reports Domain.

## Import

- validate_import()
- preview_import()
- import_clients()
- commit_import()

---

# עקרונות ארכיטקטוניים

## בעלות ישויות

- primary anchor תמיד `client_record_id`
- `business_id` הוא תיוג אופציונלי, לא בעלות
- VAT ו-Annual Reports — client-scoped בלבד, ללא business_id
- Authority Contacts — client-scoped בלבד, ללא business_id
- כל Note חייב client_record_id לצורכי permission/query

## סיווגי מס

- `vat_registration_type`, `vat_reporting_frequency`, `income_tax_type`, `national_insurance_type` — שייכים ל-LegalEntity בלבד
- Business לא מחזיק סיווגי מס
- עוסק יחיד = סיווג אחד לכל עסקיו יחד (חוק מע"מ סעיף 1)
- חברה בע"מ = תמיד עוסק מורשה מהשקל הראשון (תקנה 13(7))

## מחיקות

- ישויות ראשיות: soft_delete_* (deleted_at)
- שורות פירוט (VAT invoices, report lines, annex data): hard delete
- Contacts, Notes, Reminders, Tasks: delete_* — records תפעוליים פשוטים
- אין archive_* כ-API layer — lifecycle מנוהל דרך status + deleted_at

## cancel vs soft_delete

- cancel_* = מעבר סטטוס עסקי (הישות נשארת, הסטטוס משתנה)
- soft_delete_* = הסתרה תפעולית (deleted_at מוגדר, הישות נעלמת מתצוגה)

## הפרדת דומיינים

- אם הפעולה משנה את הלקוח עצמו → Clients Domain
- אם הפעולה רק קשורה ללקוח → דומיין אחר עם client_id
- search_* → Search Domain בלבד
- Notes → Notes Domain בלבד
- Export → Reports Domain
- Import → Client Import Domain
- Timeline → Audit Domain (aggregation, לא ישות עצמאית)
- Dashboard = read model טהור
- Work Queue = application read model שמחשב priority rules, לא משנה state
- Actions = Application Layer, לא business domain
- Health = infrastructure, לא במסמך זה
- Auth/session = core layer, לא במסמך זה

## תשלומים

- Charge = source of truth לתשלום
- Invoice = מסמך חשבונאי בלבד, לא מחזיק סטטוס תשלום
- payment events מספק חשבוניות → record_payment(charge_id, source="invoice_provider")

## תהליכי עבודה

- VAT: data_entry_in_progress → ready_for_review → filed
- Annual Report: not_started → collecting_docs → in_preparation → ready_for_accountant_review → pending_client → submitted → closed
- ready_for_review / ready_for_accountant_review: secretary + advisor
- file / submit: advisor only
- כל מעבר status נאכף ב-Service layer, לא ב-router ולא ב-frontend

## Roles

- Advisor: submit, file, finalize, approve, send_back
- Secretary: prepare, collect, mark_ready_for_review

## list_* conventions

- כל list_* תומך pagination, filters, sort
- אין list_* ללא scoping ברור (client_id / entity_id / type)

## הגדרות

- Task = עבודה שצריך לבצע
- Reminder = התראה בזמן
- Alert = signal פנימי שנוצר מחוקים עסקיים
- Notification = הודעה שנשלחת החוצה למשתמש או ללקוח

---
