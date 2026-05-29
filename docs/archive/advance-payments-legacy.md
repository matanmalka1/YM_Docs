## Scope
This file owns only:
- Archived historical content from the original advance payments specification (`backend/docs/backend/domains/advance_payments_spec.md`).

This file must not contain:
- Current implemented behavior (see canonical doc).
- New product requirements.

Source of truth: historical

# מקדמות מס הכנסה — אפיון מלא (Archived)

> Original specification document. Not the current API/model contract.
> Canonical domain doc: `docs/docs/domains/advance-payments.md`

**What was dropped and why:**
- `payment_status` field name → renamed to `status` in implementation.
- `reported_turnover` / `turnover_source_vat_report_id` fields → not implemented; current model uses `turnover_amount` (unlinked snapshot) + `TurnoverLookupRepository` for live lookups.
- `overdue` enum value → implemented as computed `timing_status` response field (correct per spec decision).
- Overview "grouped by month, collapsed" and drawer-edit UI → UI decisions, not backend; preserved as Future/planned in canonical doc.

**What was preserved (with rationale):**
- `overdue` computed, not stored — decision still valid and implemented.
- Turnover snapshot vs. live lookup design — implemented differently (no FK) but concept preserved.
- `missing_turnover` signal — implemented as read-only flag.
- `paid_late` computed field — implemented.
- `timing_status` computed field — implemented.
- Overview month-batch grouping concept — implemented via `/overview/batches`.
- Closed decisions table — carried into canonical doc §Decisions.

---

## רקע עסקי

מקדמה = תשלום תקופתי על חשבון מס הכנסה שנתי.

```
מחזור לתקופה × אחוז מקדמות (advance_rate) = סכום מקדמה
```

- due_date = ה-15 לחודש שאחרי תום התקופה
- תדירות: חודשי (period_months_count=1) או דו-חודשי (period_months_count=2)
- מחזור מגיע מדוח מע"מ של אותה תקופה — אבל אין תלות קשיחה (דוח מע"מ לא חייב להיות מוגש לפני המקדמה)
- בסוף שנה: מס שנתי סופי − מקדמות ששולמו = יתרה לתשלום / החזר

---

## מודל נתונים — שינויים נדרשים (היסטורי)

### הוצאת `overdue` מה-enum

**לפני:** `status: pending | partial | paid | overdue`

**אחרי:** שני שדות נפרדים:

| שדה | סוג | הסבר |
|-----|-----|-------|
| `payment_status` | enum: `pending \| partial \| paid` | מצב התשלום בפועל |
| `timing_status` | computed (לא נשמר) | `on_time \| overdue` — נגזר מ-`today > due_date AND payment_status != paid` |
| `paid_late` | computed (לא נשמר) | `paid_at > due_date` — שולם במלואו אבל באיחור |

> **Status:** Implemented. `status` enum is `pending|paid|partial`. `timing_status` and `paid_late` are computed response fields.

### הוספת שדות מחזור — snapshot (היסטורי, לא מומש)

| שדה | סוג | הסבר |
|-----|-----|-------|
| `reported_turnover` | `Numeric(12,2) \| null` | snapshot של מחזור בזמן דיווח/תשלום |
| `turnover_source_vat_report_id` | `Integer \| null` | FK → vat_reports — איזה דוח שימש |

> **Status:** NOT IMPLEMENTED. Current model uses `turnover_amount` (unlinked snapshot) + `TurnoverLookupRepository` for live VAT lookups.

---

## missing_turnover signal

```
missing_turnover = no vat_report for period AND no reported_turnover snapshot
```

פעולות אפשריות:
- **משוך מדוח מע"מ** — אם בינתיים נוצר דוח
- **הזן מחזור ידנית** — דיווח מקדמה ללא דוח מע"מ
- **סמן לא רלוונטי** — אין חובה בפועל

לא חוסם פעולות בודדות. **כן חוסם**: "סמן batch כמוכן לתשלום" אם יש לקוחות עם missing_turnover.

> **Status:** `missing_turnover` is a computed read flag. Batch-blocking behavior not implemented.

---

## עמוד Overview — עיצוב מוצע (היסטורי)

### גרופינג לפי חודש — collapsed כברירת מחדל

```
מקדמות מאי 2026  ·  42 לקוחות  ·  18 חסרים מחזור  ·  7 באיחור  ·  ₪128,400 לתשלום
▼ פתיחה → רשימת לקוחות
```

**header של כל batch:**
- חודש + שנה
- סה"כ לקוחות בבאטץ'
- `missing_turnover_count` — חסרי מחזור
- `overdue_count` — timing_status=overdue
- סה"כ לתשלום (expected − paid)

> **Status:** `/overview/batches` endpoint implemented returning `MonthBatchSummary` objects with these fields. UI collapsed state is a frontend concern.

---

## החלטות שנסגרו

| נושא | החלטה |
|------|--------|
| overdue | computed (timing_status), לא enum value |
| paid_late | computed מ-paid_at vs due_date |
| מחזור | live מ-vat_reports לפני תשלום, snapshot אחרי |
| עריכה | drawer (לא inline, לא modal) |
| overview layout | גרופינג לפי חודש, collapsed by default |
| advance_rate | מוצג ב-header של טאב לקוח |
| generate_schedule | נשאר בשני המסכים |
| חסר מחזור | signal תפעולי, לא חוסם — אלא אם "סמן batch כמוכן" |
