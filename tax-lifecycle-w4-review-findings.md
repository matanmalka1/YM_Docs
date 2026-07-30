# ממצאי בדיקה — Tax Lifecycle W4 Amendment

תאריך הבדיקה: 30 ביולי 2026

הבדיקה בוצעה על הענף `tax-lifecycle/w4-amendment` בשלושת הריפוזיטוריז:

- `backend`
- `frontend`
- `docs`

הבדיקה הושוותה למסמכי התכנון והמעקב:

- `docs-tax-lifecycle-refactor-plan-md-star-abundant-hoare.md`
- `docs/tax-lifecycle-refactor-plan.md`
- `docs/tax-lifecycle-refactor-progress.md`

## סיכום

הענף עדיין לא מוכן למיזוג.

נמצאה תקלה אחת שחוסמת יצירת תיקון בפועל, ועוד כמה בעיות שעלולות להסתיר דיווחים, להציג נתונים שגויים או למנוע מהמשתמש להשלים את התהליך דרך הממשק.

**מצב נכון ל-30 ביולי 2026:** ממצאים 1, 2, 3, 4, 6 ו-7 תוקנו ונכנסו לענף. ממצא 4 התברר בחקירה כשני באגים ולא אחד — פירוט בסעיף עצמו, לרבות שאלה פתוחה אחת שנשארה להכרעה. ממצא 5 פתוח. בבדיקת follow-up לאחר התיקונים נמצאו ממצאים 8–11; גם הם פתוחים, ושניים מהם מראים שממצאים 2 ו-3 עדיין אינם סגורים מקצה לקצה.

## 1. העתקת פרטי דוח שנתי נכשלת בזמן יצירת תיקון

**חומרה: חוסם — תוקן ב-30 ביולי 2026** (`3e787146`)

### מה קורה?

כאשר יוצרים תיקון לדוח שנתי, המערכת אמורה להעתיק את כל תוכן הדוח הישן לרשומה חדשה.

בזמן העתקת פרטי הדוח, הקוד משתמש בשם השדה `annual_report_id`. בפועל, שם השדה במודל הוא `report_id`.

### מה המשמעות למשתמש?

יצירת תיקון לדוח שנתי שיש בו פרטי דוח תיכשל בשגיאת שרת. המשתמש לא יוכל להתחיל את התיקון.

### מה תוקן?

- שם שדה הקישור בהעתקת ה-detail שונה ל-`report_id`. רק ה-detail חורג: `income_line`, `expense_line`, `credit_point` ו-`schedule_entry` משתמשים כולם ב-`annual_report_id`, ו-`annex_data` ב-`schedule_entry_id` — כולם נבדקו ותקינים.
- הכשל לא היה שקט: `copy_child` מעביר את מפתח האב כ-keyword דרך ה-mapper, ולכן השם השגוי הפיל את הבנייה ב-`TypeError: 'annual_report_id' is an invalid keyword argument for AnnualReportDetail` — 500 לפני שהועתקה ולו שורה אחת. כל דוח שיש בו detail (כלומר כל דוח שהוזנו בו ניכויים או הערות) לא היה ניתן לתיקון כלל.
- נוספה בדיקה אחת שמכסה את כל טבלאות הילדים יחד — `test_amendment_copies_the_whole_material`: דוח עם detail, שורת הכנסה, שורת הוצאה, נקודת זיכוי ונספח `schedule_b` מוגש, נוצר לו תיקון, וכל ילד מאומת על התיקון בזמן שהמקור שומר את שלו. אומת שהבדיקה נופלת כשמחזירים את השם השגוי.

### מיקום בקוד

- `backend/app/annual_reports/repositories/annual_report_report_repository.py:62`
- `backend/app/annual_reports/models/annual_report_detail.py:33`
- `backend/tests/annual_reports/api/test_annual_report_amendment.py` — `test_amendment_copies_the_whole_material`

## 2. מחיקת תיקון פתוח מסתירה את החיוב מכל התצוגות ומשאירה את השרשרת תקועה

**חומרה: גבוהה — תוקן ב-30 ביולי 2026**

### מה קורה?

כאשר יוצרים תיקון, הרשומה הישנה מסומנת ככזו שהוחלפה ברשומה החדשה. הרשימות הרגילות מציגות רק את הרשומה האחרונה בשרשרת.

כרגע אפשר למחוק את התיקון החדש כל עוד הוא עדיין פתוח. לאחר המחיקה:

- הרשומה הישנה מוסתרת כי היא כבר מסומנת כ"מוחלפת".
- הרשומה החדשה מוסתרת כי היא מחוקה.
- אי אפשר ליצור שוב תיקון מהרשומה הישנה, כי היא עדיין מסומנת כאילו כבר קיים לה תיקון.

הבעיה קיימת במע"מ, בדוחות שנתיים ובמקדמות.

### מה המשמעות למשתמש?

תקופה או שנת מס שלמה נעלמת מהרשימות, מהסכומים ומהדוחות, ואי אפשר להמשיך לעבוד עליה: יצירת תיקון חדש נחסמת ב-`OBLIGATION.ALREADY_AMENDED`, ויצירת רשומה מקורית חדשה נחסמת באינדקס הייחודיות — ה-predicate שלו הוא `deleted_at IS NULL AND amends_id IS NULL AND status <> 'canceled'` בלי `superseded_at`, ולכן המקור עדיין תופס את התקופה. המידע עצמו נשמר במסד הנתונים וביומן הביקורת.

### מה תוקן?

נחסמה מחיקה ישירה של רשומת תיקון, בשלושת התחומים:

- `assert_deletable` ב-`obligation_chain.py` — דוחה כל רשומה שיש לה `amends_id`, עם `OBLIGATION.AMENDMENT_NOT_DELETABLE` ו-409.
- מחובר לשלושת מסלולי המחיקה. במקדמות הוא ממוקם ב-`_soft_delete_payment`, כדי לכסות גם את ניקוי הקצב הישן (`stale cadence`) שבוחר לפי תדירות ותאריך ולא מחריג תיקונים.
- ה-409 מתועד ב-OpenAPI ובמטריצת השגיאות, וכפתור המחיקה מוסתר בשלושת ה-UI (נוסף `amends_id` ל-DTO של רשימת המע"מ למטרה זו).
- בדיקה אחת לכל תחום: יצירת תיקון ואז ניסיון מחיקה שנדחה.

### ובמקומה — פעולת ביטול תיקון מפורשת

חסימה בלבד משאירה תיקון שנפתח בטעות ללא דרך חזרה, ולכן נוספה הפעולה עצמה:
`POST /vat/work-items/{id}/withdraw`, `POST /annual-reports/{id}/withdraw`,
`POST /clients/{id}/advance-payments/{payment_id}/withdraw` — advisor בלבד.

`withdraw_amendment` ב-`obligation_chain.py` הוא הראי המדויק של `link_amendment`: שתי כתיבות שאינן נפרדות.

- מחיקה רכה של התיקון (`deleted_at` + `deleted_by`).
- ניקוי `superseded_at` מהמקור, שחוזר להיות הרשומה היחידה של התקופה.

**מחיקה רכה ולא `CANCELED`** — וזו הסיבה שאין מיגרציה: `deleted_at IS NULL` מופיע כבר גם באינדקס הייחודי על `amends_id` וגם בכל קריאת ראש-שרשרת, ולכן כתיבה אחת גם מוציאה את השורה מקבוצת הראשים וגם משחררת את הסלוט ליצירת תיקון חדש. תיקון ב-`CANCELED` היה נשאר `superseded_at IS NULL` לנצח — ראש שני קבוע לתקופה — והיה ממשיך להחזיק את הסלוט, כי האינדקס שלו אינו מחריג לפי סטטוס.

שערי `assert_withdrawable`: חייב להיות תיקון (`OBLIGATION.NOT_AN_AMENDMENT`), אסור שיהיה **סגור** (`OBLIGATION.LOCKED`), וחייב להיות ראש השרשרת (`OBLIGATION.ALREADY_AMENDED` — הגנה בלבד, כי רשומה מוחלפת היא בהכרח רשומה שהוגשה).

**`CANCELED` מותר לביטול, במכוון** — התנאי הוא "סגור" ולא "פתוח". ביטול הוא מצב סופי אבל אינו הגשה, ולכן אין רשומת הגשה להגן עליה; וחשוב מכך, תיקון מבוטל הוא אחרת מצב תקוע — הוא נשאר `superseded_at IS NULL` לנצח ולכן ממשיך להיות ראש התקופה בעוד המקור שהוגש נשאר מוסתר, ואי אפשר לא לתקן שוב ולא ליצור מחדש. `withdraw` הוא היציאה היחידה מהמצב הזה.

נעילה: המקור נטען ראשון ב-`FOR UPDATE` (id עולה, כמו ב-`create_amendment`), והשער נבדק שוב אחרי הנעילה. הבדיקה החוזרת דורשת ש-`get_by_id_for_update` **ירענן** את השורה — `SELECT` רגיל שהשורה שלו כבר ב-identity map מחזיר את האובייקט הטעון בלי לעדכן את שדותיו, ואז הבדיקה השנייה חוזרת ומאשרת בדיוק את המצב שהנעילה נועדה לא לסמוך עליו. לכן נוסף `populate_existing=True` ב-`base_repository.get_by_id_for_update` — לכל הקוראים, כי אותו באג היה חבוי בכל מסלול FOR UPDATE אחר.

היסטוריה: `select_chain` מחזיר גם תיקונים שבוטלו — רק אותם, לא כל שורה מחוקה — והם מסומנים ב-`is_withdrawn` בשלושת מודלי השרשרת ב-Frontend. `get_vat_work_item_actions` מחזיר רשימה ריקה עבור רשומה שנמשכה: היא מגיעה רק מקריאת ההיסטוריה, ואין עליה מה לעשות.

ביקורת: פעולה אחת לכל תחום, `*.amendment_withdrawn`, על התיקון, עם `restored_original_id` ב-metadata.

הערה: במקדמות ובדוחות שנתיים כפתור "בטל תיקון" לא ייראה בפועל עד שייווצר מסלול יצירת תיקון ב-Frontend (ממצא #5) — הוא מותנה ב-`amends_id != null`.

### מיקום בקוד

- `backend/app/common/obligation_chain.py` — `assert_deletable`, `assert_withdrawable`, `withdraw_amendment`, `select_chain`
- `backend/app/vat/vat_work_item_metadata.py:54` · `backend/app/vat/services/vat_amendment_service.py` — `withdraw`
- `backend/app/annual_reports/services/annual_report_service.py:102` · `annual_report_amendment_service.py` — `withdraw`
- `backend/app/advance_payments/services/advance_payment_service.py` — `_soft_delete_payment` · `advance_payment_amendment_service.py` — `withdraw`

## 3. ביטול רשומה לא באמת משחרר את התקופה

**חומרה: גבוהה — תוקן ב-30 ביולי 2026** (`6b15bc04`)

### מה קורה?

לפי החלטה D-23, רשומה שבוטלה צריכה לשחרר את התקופה או את שנת המס, כדי שאפשר יהיה ליצור רשומה חדשה במקומה.

האינדקסים במסד הנתונים אכן מאפשרים זאת. אבל לפני שמגיעים למסד הנתונים, שירותי היצירה בודקים אם קיימת רשומה כלשהי באותה תקופה. הבדיקה כוללת גם רשומה שבוטלה ולכן היצירה נעצרת מוקדם מדי.

הבעיה קיימת במע"מ, במקדמות ובדוחות שנתיים.

### מה המשמעות למשתמש?

לאחר ביטול רשומה, המשתמש עדיין לא יכול ליצור רשומה חדשה לאותה תקופה או שנה. מבחינתו הביטול לא באמת שחרר את המקום.

### מה תוקן?

השאלה "האם התקופה תפוסה?" הופרדה מהשאלה "איזו שורה התקופה מציגה?", ושתיהן קיבלו שאילתה בשם ב-`obligation_chain.py`:

- `select_slot_occupant` — ה-predicate של האינדקס החלקי, מילה במילה: לא מחוקה, לא תיקון, לא `canceled`. **השאילתה היחידה ששער יצירה רשאי לשאול.** נבנית עם `include_superseded=True` במכוון, כי מקור שהוחלף כן ממשיך להחזיק את הסלוט שלו.
- `select_current_obligation` — השורה התפעולית: ממוינת ולא מסוננת, כך שרשומות מבוטלות נדחקות לסוף אבל אינן נעלמות, ותקופה שכל שורותיה מבוטלות עדיין מוצגת כמבוטלת. המיון נושא משקל: `.first()` בלי מיון בחר לפי תוכנית השאילתה — sequential scan החזיר את המבוטלת, index scan את החיה.

שני שערי היצירה בשלושת התחומים, שתי לולאות ה-onboarding sync ו-`_generate_year_for_client` הועברו לשאלת הסלוט. `AdvancePaymentRepository.exists_for_period` נמחקה במקום לתקן אותה — שני הקוראים שלה היו שערי יצירה, ולכן היא הפכה ל-`get_slot_occupant_for_period` ושלושת התחומים שואלים אותו דבר.

אין שינוי סכימה ואין שינוי חוזה: האינדקסים היו נכונים מלכתחילה.

### מיקום בקוד

- `backend/app/common/obligation_chain.py` — `select_slot_occupant`, `select_current_obligation`
- `backend/app/vat/services/vat_intake_service.py`
- `backend/app/advance_payments/services/advance_payment_service.py`
- `backend/app/annual_reports/services/annual_report_create_service.py`
- `backend/tests/common/test_obligation_slot_occupancy.py`

## 4. האיחור של התקופה נמחק בהגשת התיקון, ודוח הציות מחשב אותו מחדש במקום לקרוא אותו

**חומרה: גבוהה — תוקן ב-30 ביולי 2026** (`dc2038c3`)

בחקירה לעומק נמצא שאלה **שני באגים**, לא אחד. התיקון שהומלץ כאן במקור (לקרוא את `chain_closed_late` בדוח) לא היה פותר את הבעיה בפני עצמו, כי הערך כבר NULL בשלב שבו הדוח קורא אותו.

### 4a. `chain_closed_late` נדרס ל-NULL כשהתיקון מוגש

`link_amendment` מעביר את האיחור של התקופה אל התיקון בזמן הלידה, וזה עובד. אבל בהגשת התיקון, `record_closing_lateness` כותב את **שני** השדות ללא תנאי:

```python
record.closed_late = closed_late
record.chain_closed_late = closed_late
```

לתיקון אין תאריך יעד (D-14), ולכן `compute_closed_late(closed_at, None)` מחזיר `None` — והשורה השנייה מוחקת את מה שהועבר בלידה.

הבעיה בשלושת התחומים, ושניים מהם אפילו לא עוברים דרך ה-helper אלא בונים dict ידנית (`update_fields["chain_closed_late"]` בדוחות שנתיים, `fields["chain_closed_late"]` במקדמות) — ולכן תיקון ב-helper בלבד לא יסגור אותם.

### 4b. הדוח משחזר את האיחור במקום לקרוא אותו

שאילתת הדוח מביאה `closed_at` ו-`due_date_effective` ולא את שדות האיחור כלל, והדוח משווה ביניהם. `deadline is None` → `continue`, והתקופה אינה נספרת לא כדיווח בזמן ולא כדיווח באיחור.

זו גם הפרה של D-20 עצמו, גם בלי תיקונים: האיחור הוא עובדה שנכתבת פעם אחת בסגירה, ואסור לגזור אותו מקריאה מאוחרת של תאריך יעד.

### מה מוכיח את זה

repro על תקופה `2026-01` (מע"מ), מול הענף כפי שהוא:

```
ORIGINAL       due=2026-02-16  closed_at=2026-07-30  closed_late=True  chain=True
amend          201
AT BIRTH       amendment due=None  closed_late=None  chain=True      ← הועבר נכון
REPORT (open)  expected=1 filed=0 on_time=0 late=0 rate=0.00
AFTER FILING   amendment closed_late=None  chain=None                ← ★ נמחק
REPORT (filed) expected=1 filed=1 on_time=0 late=0 rate=100.00
```

### מה המשמעות למשתמש?

- דיווח שבוצע באיחור נעלם מספירת האיחורים אחרי שהוגש לו תיקון. `on_time_count + late_count` קטן מ-`periods_filed` ואף בדיקה לא נכשלת.
- הנזק אינו בדוח בלבד. ב-Frontend `chain_closed_late` הוא הקורא היחיד של העובדה הזו, בכרטיסי המקדמות — badge "שולם באיחור" פשוט נעלם אחרי תיקון. הוא אינו הופך לירוק שקרי, כי `timing_status` יוצא `not_applicable` בהיעדר תאריך יעד, אבל המידע אבד גם ב-UI.
- ב-Backend אין היום שום קורא של `chain_closed_late` מלבד ה-DTOs. הצרכן הלוגי היחיד הוא דוח הציות, והוא לא קורא אותו — כלומר העמודה שנוספה ב-W4 היא כרגע write-only.

### מה תוקן?

1. **הכלל עבר למקום אחד** — `closing_lateness_fields` ב-`obligation_chain.py` מחזירה את שתי העמודות שסגירה כותבת, ועל תיקון משאירה את `chain_closed_late` כפי שהועבר בלידה. `record_closing_lateness` מציבה את מה שהיא מחזירה.
2. **שני העוקפים חוסלו** — הפונקציה מחזירה fields ולא מציבה, בדיוק מפני שהדוחות השנתיים והמקדמות סוגרים דרך dict שמועבר ל-`repo.update`. לכל אחד מהם היה עותק משלו של הצמד, וזו הסיבה שהכלל נשבר בשני מקומות בבת אחת. שניהם קוראים לה עכשיו.
3. **הדוח קורא במקום לחשב** — השאילתה מביאה `closed_late` ו-`chain_closed_late` במקום `closed_at` ו-`due_date_effective`, והדוח מסווג לפי תשובת השרשרת, עם נפילה חזרה לתשובת השורה עצמה. `continue` רק כששניהם NULL: תקופה שמעולם לא היה לה תאריך יעד אינה "בזמן".
4. **seed** — שלושת ה-builders עוברים ב-helper המשותף. seed המע"מ בונה תיקונים, והם ירשו את הפער.

אין שינוי סכימה ואין שינוי חוזה: שתי העמודות כבר היו קיימות וכבר היו על ה-DTOs.

בדיקות: `tests/common/test_chain_closing_lateness.py` — האיחור שורד את סגירת התיקון בשלושת התחומים, כל אחד דרך מסלול הסגירה האמיתי שלו. `tests/reports/api/test_vat_compliance_lateness.py` — תקופה מתוקנת עדיין נספרת באיחור, האינווריאנטה `on_time + late == periods_filed`, ותקופה בלי תאריך יעד שאינה נספרת באף דלי. כל אחד משני הבאגים הוחזר בנפרד כדי לאמת שהבדיקות נופלות מהסיבה הנטענת.

### שאלה פתוחה — לא באג, חוסר החלטה

**מה נחשבת "תקופה שהוגשה" כשיש עליה תיקון פתוח?**

לקוח דיווח מע"מ לינואר, הדיווח הוגש, והדוח מראה 1 מתוך 1 — 100%. חודש אחר כך מתגלה טעות ונפתח תיקון. התיקון נולד פתוח (D-21), ומאותו רגע הדוח מראה **0 מתוך 1 — 0%**, עד שהתיקון יוגש. ראו את שורת `REPORT (open)` ב-repro למעלה.

זה נובע מ-D-12 — השרשרת מיוצגת בשורה אחת, האחרונה — שפוגש `filed_case = status == SUBMITTED`. השורה האחרונה היא התיקון, והתיקון עדיין לא הוגש. אין לזה D-number, והקוד עושה בדיוק את מה שנכתב.

שתי אפשרויות:

- **א. הראש קובע** — כפי שקורה היום. התקופה נחשבת לא-מוגשת כל עוד יש עליה עבודה פתוחה. הצד החזק: אין רגע שבו הדוח אומר "הוגש" על שורה שאינה מוגשת, גם אם התיקון יבוטל בהמשך.
- **ב. "הוגש" נמדד ברמת השרשרת** — התקופה נשארת מוגשת, כי מישהו כבר הגיש אותה לרשויות ותיקון בעבודה לא מבטל את העובדה הזו. הצד החזק: מדד ציות אמור לענות "האם המשרד הגיש בזמן" ולא "האם יש כרגע עבודה פתוחה" — לשאלה השנייה כבר יש עמודה משלה, `periods_open`.

**המלצה: ב.** אבל היא דורשת עמודה שנושאת את התשובה, `chain_closed_at` באותה תבנית בדיוק כמו `chain_closed_late` — אחרת כל קריאה תצטרך ללכת אחורה בשרשרת, וזה מה ש-D-12 קיים כדי למנוע. זו עבודה גדולה מממצא 4, ולכן היא מופרדת ממנו ולא נכללה בתיקון.

בינתיים ההשלכה בפועל: אחוז הציות של לקוח יורד ל-0% לאורך כל הזמן שהתיקון פתוח, וחוזר כשהוא מוגש.

### מיקום בקוד

- `backend/app/common/obligation_chain.py` — `record_closing_lateness`, `link_amendment`
- `backend/app/common/obligation_closing.py` — `compute_closed_late`
- `backend/app/vat/repositories/vat_work_item_write_repository.py` — `mark_filed`
- `backend/app/annual_reports/services/annual_report_status_service.py` · `backend/app/advance_payments/services/advance_payment_service.py` — שני העוקפים
- `backend/app/vat/repositories/vat_compliance_repository.py` — `get_filed_items_for_clients`
- `backend/app/reports/vat_compliance_report.py` — סיווג on-time/late
- `backend/app/seed/builders/demo/{vat,reports,advance_payments}.py`
- `frontend/src/features/advancedPayments/components/panel/AdvancePaymentContextCard.tsx` · `.../clientAdvancePayments/ClientAdvancePaymentsCards.tsx`

## 5. תהליך התיקון ב-Frontend הושלם רק עבור מע"מ

**חומרה: גבוהה — תוקן ב-30 ביולי 2026**

### מה קורה?

ב-Backend קיים endpoint ליצירת תיקון בכל שלושת התחומים. ב-Frontend:

- במע"מ קיימת פעולת יצירת תיקון.
- בדוחות שנתיים קיימת רק צפייה בהיסטוריית השרשרת.
- במקדמות קיימת רק צפייה בהיסטוריית השרשרת.

כלומר, אין דרך ליצור תיקון לדוח שנתי או למקדמה דרך הממשק.

גם במע"מ, לאחר יצירת התיקון הקוד רק מרענן את הנתונים ולא מעביר את המשתמש לרשומת התיקון החדשה, אף שההערה בקוד אומרת שזה התהליך הרצוי.

בנוסף, רשימות המע"מ והדוחות השנתיים לא מקבלות את שדות השרשרת ולכן אינן יכולות לסמן למשתמש שהרשומה המוצגת היא תיקון.

### מה המשמעות למשתמש?

המשתמש לא יכול להשלים את תהליך W4 בשני תחומים מתוך שלושה. במע"מ הוא עלול להישאר על הרשומה הישנה והנעולה במקום לעבור לתיקון החדש.

### מה תוקן?

**פעולת "צור תיקון" בדוחות שנתיים ובמקדמות.** endpoint, פונקציית API, mutation, כפתור ומצב טעינה. השער הוא ההפך מכל פעולה אחרת באותם מסכים — הרשומה חייבת להיות **מוגשת**, לא פתוחה — ולכן הוא אינו יכול לשבת בתוך `canMutateReport` או `canEdit`, ששניהם שוללים `submitted`. התנאי השני, `superseded_at == null`, הוא החצי השני של `assert_amendable`: שרשרת היא קו, ולרשומה שכבר יש לה תיקון אין תיקון שני לתת. במקדמות בלוק הפעולות כולו היה מותנה ב-`canEdit`, שהוא `false` בדיוק על הרשומות שהפעולה מוצעת עליהן — לכן הותנה מחדש ב-`canEdit || onCreateAmendment`, ושמירה נשארה מאחורי `canEdit`.

**מעבר אוטומטי לרשומה החדשה, בשלושת התחומים.** הרשומה שנעזבת נעולה (D-13), ולכן היישארות עליה מותירה את המשתמש במסך היחיד שבו אי אפשר להזין את הנתונים המתוקנים. הניווט הוא `push` ולא `replace` — בניגוד לביטול תיקון — כי המקור עדיין קיים ו-Back הוא מקום אמיתי לחזור אליו.

**`toSiblingRecordPath`** ריכז את החלפת מקטע המזהה שכל שישה אתרי הניווט חולקים. כל פאנל מותקן פעמיים — מסך עצמאי וכרטיס לקוח — ולכן שם של route היה מפיל מבקר client-scoped מחוץ ללשונית, לפירורי הלחם ולניווט שלה.

**פתיחת רשומות קודמות מהשרשרת.** רשומה מוחלפת ניתנת לשליפה לפי id בדיוק בשביל הקריאה הזו, ולכן זו הייתה עבודת frontend בלבד. שתי שורות נשארו ללא קישור במכוון: הרשומה שכבר על המסך, ותיקון **שבוטל** — הוא נמחק רכות, וקריאת הפרטים שלו מחזירה 404.

**סימון "תיקון" ברשימות.** `amends_id` כבר הוחזר ב-`VatWorkItemListItem` (ממצא 2 הוסיף אותו לשער המחיקה) ולכן במע"מ זה היה frontend בלבד; ל-`AnnualReportListItem` ול-`AdvancePaymentOverviewRow` הוא נוסף. השדה אינו נתון שהרשימה מציגה — שרשרת מוצגת כשורה אחת בכל מקום (D-12), ולכן בלעדיו תיקון והרשומה שהוא החליף אינם נבדלים באף תא. בטבלת העונה הסימון יושב על עמודת הסטטוס: היא שורה אחת ללקוח לשנת מס אחת, ולכן שום תא אחר אינו יכול להיבדל.

**17 בדיקות Frontend** — בניית endpoint ומיפוי תשובה לשתי קריאות ה-`createAmendment` החדשות, הניווט שכל תחום מבצע, והשער שקובע אם הפעולה קיימת בכלל. בדיקת הניווט של המקדמות אומתה כרגרסיה: היא נופלת על ה-`${backPath}/${id}` שלפני התיקון.

**ממצאים 9 ו-10 נסגרו יחד עם זה** — הם נוגעים באותם קבצים, ותיקון ממצא 5 בלעדיהם היה מוסיף ניווט תקין לצד ניווט שבור.

### מיקום בקוד

- `frontend/src/utils/recordPath.ts` (חדש) · `recordPath.test.ts`
- `frontend/src/features/annualReports/api/annualReports.api.ts` · `hooks/useReportMutations.ts` · `components/panel/AnnualReportFullPanel.tsx`
- `frontend/src/features/advancedPayments/api/advancedPayments.api.ts` · `hooks/useAdvancePaymentDetailPage.ts` · `components/panel/AdvancePaymentDetailView.tsx`
- `frontend/src/features/vatReports/hooks/useCreateVatAmendment.ts` · `hooks/useVatInvalidation.ts`
- שלושת מודלי השרשרת: `VatChainModal.tsx` · `AnnualReportChainModal.tsx` · `AdvancePaymentChainModal.tsx`
- `backend/app/annual_reports/schemas/annual_report_responses.py` — `AnnualReportListItem.amends_id`
- `backend/app/advance_payments/schemas/advance_payment.py` · `api/advance_payment_routes_overview.py` — `AdvancePaymentOverviewRow.amends_id`

## 6. יצירת תיקון למקדמה אינה מוגנת מפני שתי בקשות במקביל

**חומרה: בינונית — תוקן ב-30 ביולי 2026**

### מה קורה?

במע"מ ובדוחות שנתיים, יצירת תיקון נועלת את הרשומה המקורית בזמן הבדיקה והיצירה. במקדמות, הרשומה נקראה בלי נעילה.

אם שני משתמשים או שתי בקשות ינסו ליצור תיקון לאותה מקדמה כמעט באותו זמן, שתיהן יכולות לעבור את בדיקת "עדיין אין תיקון".

### מה המשמעות למשתמש?

אחת הבקשות עלולה לקבל שגיאת שרת לא ברורה במקום הודעת קונפליקט מסודרת.

### מה תוקן?

- `create_amendment` במקדמות קורא כעת את המקורית פעמיים, בדיוק כמו `withdraw` שלידו: קודם דרך `get_payment_for_client` — שהוא היחיד שבודק שייכות ללקוח, כי הנעילה נלקחת לפי מפתח ראשי ואינה בודקת בעלות — ואחר כך דרך `repo.get_by_id_for_update` לפני `assert_amendable`. הקריאה הנעולה מגיעה עם `populate_existing`, ולכן היא דורסת את השורה שהקריאה הראשונה הכניסה ל-identity map: השער נבדק מול מה שיש במסד אחרי ההמתנה לנעילה, לא מול מה שנקרא לפניה.
- **התוצאה בפועל חמורה יותר ממה שהממצא תיאר.** אומת שבלי הנעילה הבקשה השנייה אינה נופלת על מגבלת הייחודיות בכלל אלא **מצליחה ומחזירה 201** — האינדקס `uq_advance_payment_amends` שומר על `amends_id` הייחודי, ושתי הבקשות מייצרות שני תיקונים לאותה מקורית רק אם שתיהן נצמדות לאותו `amends_id`; בתרחיש שנבדק התוצאה היא תקופה עם שני ראשי שרשרת ולא 500. עם הנעילה מוחזר 409 `OBLIGATION.ALREADY_AMENDED`.
- **טיפול ב-`IntegrityError` לא נוסף, במכוון.** עם הנעילה הכתיבה השנייה מחכה, ואחריה `assert_amendable` רואה `superseded_at` ומחזירה 409 לפני ה-INSERT — כך שהמגבלה אינה ניתנת להפעלה מהמסלול הזה. האינדקס השני, `uq_advance_payment_client_record_period_active`, מחריג `amends_id IS NOT NULL` ולכן אינו רלוונטי ליצירת תיקון. מע"מ ודוחות שנתיים אינם תופסים `IntegrityError` בנתיב הזה גם הם; מטפל ייעודי במקדמות בלבד היה מפצל את התבנית המשותפת עבור מצב שלא ניתן להגיע אליו.
- נוספה בדיקה שמדמה את המרוץ: `test_a_correction_that_lands_before_the_lock_stops_the_second_one`. הסוויטה רצה על חיבור אחד בתוך טרנזקציה אחת, ולכן מרוץ אמיתי בשתי טרנזקציות לא ניתן לביים — הבדיקה מייצרת את אותה סטייה ישירות, באותה שיטה שכבר משמשת ב-`test_a_filing_that_lands_before_the_lock_stops_the_withdrawal` במע"מ: המקורית טעונה ב-identity map עם `superseded_at IS NULL`, ו-UPDATE ב-SQL גולמי מזיז את המסד תחתיה. אומת שהבדיקה נופלת (201 במקום 409) כשמסירים את הנעילה.

### מיקום בקוד

- `backend/app/advance_payments/services/advance_payment_amendment_service.py:32`
- `backend/tests/advance_payments/api/test_advance_payment_amendment.py` — `test_a_correction_that_lands_before_the_lock_stops_the_second_one`

## 7. מסמכי מקור האמת ומסמך ההתקדמות לא עודכנו

**חומרה: בינונית — תוקן ב-30 ביולי 2026**

### מה קורה?

מסמכי התחומים עדיין מתארים חלק מההתנהגות הישנה:

- מסמך מע"מ עדיין מתאר יצירת תיקון בזמן ההגשה בעזרת `is_amendment` ו-`amends_item_id`.
- מסמך הדוחות השנתיים עדיין אומר שתיקון פותח מחדש את אותה רשומה.
- מסמך ההתקדמות עדיין מציג את W4 כעבודה פתוחה, ומציין מזהה migration ישן.

### מה המשמעות?

מפתח או סוכן שיקראו את התיעוד יקבלו תמונה שונה מהקוד. הדבר מגדיל את הסיכוי לבאגים, לתיקונים שגויים ולבדיקות שמאמתות התנהגות שכבר אינה רצויה.

### מה תוקן?

- שלושת מסמכי התחומים עודכנו ל-`ObligationStatus` המשותף ולשרשרת תיקונים כרשומות נפרדות עם `amends_id`, `superseded_at`, `chain_closed_late`, יצירה, ביטול וקריאת היסטוריה.
- תיאורי `is_amendment`/`amends_item_id`, פתיחה מחדש של אותה רשומת דוח שנתי, enums מקומיים שפרשו ומסלולי קוד ששמם השתנה הוסרו או תוקנו.
- מסמך ההתקדמות מתאר את גבול W4 האמיתי: תשתית ה-Backend קיימת בשלושת התחומים; יצירת תיקון ב-Frontend קיימת כרגע רק במע"מ; ה-seed יוצר שרשרת הדגמה רק במע"מ; ממצאים 5 ו-8–11 נשארו פתוחים.
- מזהה ה-migration עודכן ל-`b64502fa7c50` ולקובץ `b64502fa7c50_initial.py`, ומפת הענפים תוקנה כך ש-W0–W3 מתועדים על `main` ו-W4 על `tax-lifecycle/w4-amendment`.
- מסמכי התחומים אומתו מחדש מול המודלים, השירותים, ה-repositories, `backend/openapi.json`, ה-Frontend, ה-seed ושמות קובצי הבדיקות. זהו תיקון תיעוד בלבד; לא הורצו בדיקות מחדש.

### מיקום בתיעוד

- `docs/domains/vat.md`
- `docs/domains/advance-payments.md`
- `docs/domains/annual-reports.md`
- `docs/tax-lifecycle-refactor-progress.md`

## 8. ביטול ויצירה מחדש משאירים שתי רשומות תפעוליות ברשימות ובאגרגציות

**חומרה: גבוהה**

### מה קורה?

תיקון ממצא 3 מאפשר כעת מצב חוקי לפי D-23: רשומה ישנה באותה תקופה נמצאת ב-`canceled`, ולאחריה נוצרת רשומה מקורית חדשה ופעילה.

`select_current_obligation` בוחר נכון את הרשומה הפעילה בקריאה נקודתית. אבל `select_obligations`, שעליו מבוססות הרשימות, הספירות והסכומים, מסנן רק:

- `deleted_at IS NULL`
- `superseded_at IS NULL`

גם הרשומה המבוטלת וגם הרשומה החדשה עונות על שני התנאים. לכן שתיהן נכנסות לכל קריאה רחבה, אף שלתקופה אמורה להיות רשומה תפעולית אחת.

התוצאה מופיעה בשלושת התחומים:

- סיכומי מע"מ סוכמים את שתי הרשומות ומגדילים את `periods_count`.
- KPI של מקדמות סוכמים את הסכום הצפוי והסכום ששולם של שתיהן.
- סיכום עונת הדוחות השנתיים סופר את אותו לקוח ושנת מס פעמיים, בסטטוסים שונים.
- דוח הציות במע"מ יכול לספור שתי תקופות צפויות עבור חודש אחד. אם החדשה הוגשה והישנה מבוטלת, הדוח עשוי להציג `1 מתוך 2` ו-50% במקום `1 מתוך 1` ו-100%.

אותו פער קיים ב-Frontend: שלושת חלונות השרשרת מציגים badge של "נוכחי" לכל רשומה שאינה משוכה וש-`superseded_at` שלה NULL. לאחר `canceled → create fresh`, גם הרשומה המבוטלת וגם החדשה מסומנות כנוכחיות.

### מה המשמעות למשתמש?

- רשומות כפולות ברשימות עבור אותה תקופה או שנת מס.
- סכומי מע"מ ומקדמות שגויים.
- ספירות סטטוס ו-KPI שגויות.
- שיעור ציות מע"מ שגוי.
- שתי רשומות שמוצגות בו-זמנית כ"נוכחיות" בהיסטוריה.

### מה צריך לתקן?

- להגדיר scope משותף של "הרשומה התפעולית לכל תקופה": להעדיף רשומה שאינה מבוטלת; אם אין כזו, לבחור את הרשומה המבוטלת האחרונה.
- להשתמש ב-scope הזה בכל הרשימות, הספירות והאגרגציות שמציגות מצב נוכחי.
- להשאיר את `select_chain` כקריאת היסטוריה שמחזירה את כל הניסיונות; אסור לפתור את הבעיה באמצעות הסתרה גורפת של רשומות מבוטלות, כי תקופה שיש לה רק רשומה מבוטלת עדיין צריכה להיות מוצגת.
- לקבוע את badge ה"נוכחי" לפי הרשומה התפעולית שנבחרה, ולא לפי `superseded_at == null` בלבד.
- להוסיף בדיקות לכל שלושת התחומים עבור `canceled → create fresh`, כולל רשימה, ספירה, סכום ושרשרת.

### מיקום בקוד

- `backend/app/common/obligation_chain.py:95` — `chain_tip_clause`
- `backend/app/common/obligation_chain.py:111` — `select_obligations`
- `backend/app/common/obligation_chain.py:202` — `select_current_obligation`
- `backend/app/vat/repositories/vat_client_summary_repository.py:120`
- `backend/app/vat/repositories/vat_compliance_repository.py:19`
- `backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py:259`
- `backend/app/annual_reports/repositories/annual_report_report_lifecycle_repository.py:117`
- `frontend/src/features/vatReports/components/shared/VatChainModal.tsx:53`
- `frontend/src/features/annualReports/components/shared/AnnualReportChainModal.tsx:54`
- `frontend/src/features/advancedPayments/components/panel/AdvancePaymentChainModal.tsx:63`

## 9. ביטול תיקון מקדמה מהמסך העצמאי מנתב לכתובת שאינה קיימת

**חומרה: גבוהה — תוקן ב-30 ביולי 2026** (יחד עם ממצא 5)

### מה קורה?

לאחר ביטול תיקון מקדמה, `useAdvancePaymentDetailPage` מנווט אל:

```text
${backPath}/${original.id}
```

במסך העצמאי `backPath` הוא `/tax/advance-payments${location.search}`. אבל נתיב הפרטים הרשום ב-router הוא:

```text
/tax/advance-payments/:clientId/:paymentId
```

לכן, ללא query string, הניווט הוא אל `/tax/advance-payments/:originalId` וחסר בו `clientId`. עם query string, מזהה המקור אף משורשר אחרי ה-query ונוצרת כתובת משובשת.

במסך המקדמות שבתוך כרטיס הלקוח ה-`backPath` שונה ולכן המסלול הזה עשוי לעבוד; הכשל הוא במסך העצמאי.

### מה המשמעות למשתמש?

פעולת הביטול מצליחה בשרת, אבל מיד אחריה המשתמש מגיע למסלול לא קיים במקום לרשומת המקור ששוחזרה.

### מה תוקן?

- הנתיב נבנה מה-`pathname` הנוכחי ולא מ-`backPath`, דרך `toSiblingRecordPath` — אותו helper שממצא 5 הוסיף לכל שישה אתרי הניווט. הוא מקבל pathname בלבד, ולכן query string אינו יכול להגיע לתוך הנתיב.
- הטעות עצמה היא ההנחה ש-`backPath` הוא נתיב-אב של מסך הפרטים. במסך העצמאי הוא נתיב **הרשימה**, ומזהה הלקוח יושב בו במקום אחר.
- בדיקה לשני mounting contexts, עם ובלי query string. אומת שהיא נופלת על `${backPath}/${original.id}` ועוברת על התיקון.

### מיקום בקוד

- `frontend/src/utils/recordPath.ts` (חדש)
- `frontend/src/features/advancedPayments/hooks/useAdvancePaymentDetailPage.ts` — `onWithdrawSuccess`
- `frontend/src/features/advancedPayments/hooks/useAdvancePaymentDetailPage.test.tsx`

## 10. יצירת תיקון וביטול תיקון במע"מ אינם מרעננים את cache השרשרת

**חומרה: בינונית — תוקן ב-30 ביולי 2026** (יחד עם ממצא 5)

### מה קורה?

`invalidateVatWorkItem` מרענן רשימות, detail, חשבוניות, audit וסיכומי לקוח, אבל אינו מרענן את `vatReportsQK.chain(id)`.

גם `useCreateVatAmendment` וגם `useWithdrawVatAmendment` משתמשים ב-helper הזה בלבד. לכן שרשרת שכבר נטענה יכולה להישאר ב-cache אחרי:

- יצירת תיקון חדש.
- ביטול תיקון ושחזור המקור.

בשני התחומים האחרים הביטול מרענן את כל מרחב המפתחות של התחום, ולכן השרשרת מכוסה.

### מה המשמעות למשתמש?

פתיחה חוזרת של חלון ההיסטוריה יכולה להציג שרשרת ישנה: תיקון חדש שאינו מופיע, תיקון שבוטל שעדיין מוצג כפעיל, או `superseded_at` ישן.

### מה תוקן?

- `invalidateVatWorkItem` קיבל `includeChain`, **כבוי כברירת מחדל**: מפתח השרשרת יושב מחוץ למרחב הרשימות וה-detail, ולכן שינוי סטטוס רגיל אינו יכול לגעת בו. רק יצירה וביטול של תיקון כותבים מחדש את השרשרת, ורק הם מדליקים את הדגל.
- שני ה-hooks מבטלים את שתי השרשראות — של המקור ושל התיקון — ומחכים ל-invalidation לפני הניווט, כדי שהמסך הבא לא ייטען על cache ישן.
- בדיקה שמאמתת ששני מפתחות השרשרת בוטלו ביצירת תיקון.

הערה: בדוחות שנתיים ובמקדמות `chain` נמצא תחת `annualReportsQK.all` / `advancedPaymentsQK.all`, ולכן ה-invalidation הגורף שם כיסה אותו מלכתחילה. הפער היה ייחודי למע"מ.

### מיקום בקוד

- `frontend/src/features/vatReports/hooks/useVatInvalidation.ts` — `includeChain`
- `frontend/src/features/vatReports/hooks/useCreateVatAmendment.ts` · `useWithdrawVatAmendment.ts`
- `frontend/src/features/vatReports/hooks/useCreateVatAmendment.test.tsx`

## 11. שערי יצירת רשומה מקורית אינם אטומיים ועלולים להחזיר 500

**חומרה: בינונית**

### מה קורה?

ביצירת מע"מ, דוח שנתי או מקדמה השירות מבצע שתי פעולות נפרדות:

1. `select_slot_occupant` כדי לבדוק שהתקופה פנויה.
2. `INSERT` של הרשומה החדשה.

אין נעילה או מנגנון אחר שמסדר שתי יצירות לאותה תקופה, ואין טיפול ב-`IntegrityError` של האינדקס הייחודי. שתי בקשות מקבילות יכולות לראות סלוט פנוי; אחת תצליח והשנייה תיכשל מול האינדקס ותגיע ל-`database_exception_handler`, שמחזיר 500.

זה אינו ממצא 6: ממצא 6 עוסק ביצירת **תיקון מקדמה**. הממצא הזה עוסק ביצירת **רשומה מקורית** בשלושת התחומים.

### מה המשמעות למשתמש?

בקשה מתחרה מקבלת "שגיאה לא צפויה" במקום 409 ברור שמסביר שהתקופה כבר נתפסה.

### מה צריך לתקן?

- לתרגם הפרת unique רלוונטית ל-`ConflictError` בתוך savepoint, כך שה-session יישאר שמיש; או לסדר את היצירות באמצעות נעילת האב או advisory lock.
- להשאיר את `select_slot_occupant` כהודעת conflict מוקדמת, אך לא לסמוך עליו כהגנה מפני race.
- להוסיף בדיקות עם שני sessions לכל שלושת התחומים.

### מיקום בקוד

- `backend/app/vat/services/vat_intake_service.py:66`
- `backend/app/annual_reports/services/annual_report_create_service.py:85`
- `backend/app/advance_payments/services/advance_payment_service.py:497`
- `backend/app/core/exception_handlers.py:76`

## בדיקות שבוצעו במהלך הבדיקה

- שלושת הענפים היו נקיים לפני יצירת מסמך זה.
- בדיקת Ruff של ה-Backend עברה.
- בדיקת התאמת OpenAPI עברה.
- `npm run check` ב-Frontend עבר.
- עברו 51 קובצי בדיקה ו-173 בדיקות Frontend.
- בבדיקות Backend ממוקדות עברו 510 בדיקות, אך 6 בדיקות נעצרו בשלב ההכנה מפני שמסד הנתונים המקומי לבדיקות אינו מעודכן וחסר בו `client_office_number_seq`.
- לא בוצע reset למסד הנתונים במסגרת review זה.
- לא נמצאו בדיקות ייעודיות שמכסות את תהליך התיקון המלא בדוחות שנתיים ובמקדמות, או את ההשפעה של תיקונים על כל הסכומים המצטברים.

עדכון 30 ביולי 2026, אחרי תיקוני 1 ו-3:

- `pytest tests/annual_reports tests/common` — 205 עברו, כולל הבדיקה החדשה של העתקת החומר ובדיקות תפוסת הסלוט.
- `ruff format` ו-`ruff check` על הקבצים שנגעו — נקי.
- אין ולו בדיקה אחת בכל ה-Backend שמזכירה את `chain_closed_late` (ממצא 4).

עדכון follow-up לאחר תיקוני 1–4:

- סוויטת Backend מלאה — 2,362 עברו, 1 דולגה.
- בדיקות הרגרסיה החדשות של ממצא 4 — 6 עברו.
- Frontend — 51 קובצי בדיקה ו-174 בדיקות עברו.
- TypeScript, ESLint, בדיקות הארכיטקטורה, Knip, Ruff, Pyright וסנכרון OpenAPI עברו.
- ההצלחות אינן מכסות את ממצאים 8–11: אין בדיקה לרשימות ולאגרגציות אחרי `canceled → create fresh`, לניווט אחרי ביטול מקדמה, לרענון cache השרשרת במע"מ או לשתי יצירות מקור מקבילות.

## המלצת סדר תיקון

1. ~~לתקן את העתקת פרטי הדוח השנתי.~~ תוקן — ה-detail מועתק לפי `report_id`, עם בדיקה שמכסה את כל טבלאות הילדים.
2. ~~למנוע מחיקת תיקון פתוח או להחזיר נכון את הרשומה הקודמת.~~ תוקן — מחיקה נחסמה, ונוספה פעולת ביטול תיקון אטומית.
3. ~~לתקן יצירה מחדש אחרי ביטול.~~ תוקן — שערי היצירה שואלים את `select_slot_occupant`, וקריאה תפעולית שואלת את `select_current_obligation`.
4. ~~לתקן את האיחור: קודם להפסיק למחוק את `chain_closed_late` בהגשת התיקון (4a), ורק אז להסב את דוח הציות לקרוא אותו (4b).~~ תוקן — הכלל רוכז ב-`closing_lateness_fields`, והדוח קורא את העובדה במקום לגזור אותה מחדש.
5. להשלים את תהליך ה-Frontend בכל שלושת התחומים.
6. ~~להוסיף נעילה ליצירת תיקון מקדמה.~~ תוקן — המקורית נקראת תחת `FOR UPDATE` לפני השער, כמו בשני התחומים האחרים, עם בדיקה שמדמה את המרוץ.
7. ~~לעדכן את מסמכי מקור האמת וההתקדמות.~~ תוקן — שלושת מסמכי התחומים ומסמך ההתקדמות משקפים את היישום ואת גבול W4 בפועל. בדיקות הרגרסיה הנדרשות לממצאים הפתוחים נשארות בסעיפים שלהם.
8. לתקן את scope הרשומה התפעולית ברשימות ובאגרגציות לפני המשך עבודת ה-Frontend — אחרת המסכים החדשים ייבנו על ספירות וסכומים שגויים.
9. לתקן את הניווט אחרי ביטול תיקון מקדמה ואת invalidation שרשרת המע"מ.
10. לטפל ב-race של יצירת רשומה מקורית מממצא 11. ממצא 6 נסגר בנפרד ובנעילה בלבד: שם הנעילה נלקחת על שורה שכבר קיימת (המקורית), ולכן היא מספיקה; ביצירת רשומה מקורית אין שורה כזו, ולכן שם נדרש תרגום הפרת unique ל-409 או advisory lock.
