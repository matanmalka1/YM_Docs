# תכנון ביצוע: חיפוש גלובלי — שלב 1

## מה לעשות

לתכנן את הביצוע של שלב 1 מאפיון החיפוש הגלובלי, ואז לממש אותו issue אחר issue.
**התחל בתוכנית** — סדר עבודה, פירוק לקומיטים, נקודות סיכון — והצג אותה לאישור לפני קוד.

האפיון סגור. כל ההכרעות (D1–D8) התקבלו ומתועדות. אל תפתח אותן מחדש ואל תציע חלופות
למה שהוכרע — אם מצאת סתירה בין האפיון לקוד, עצור ודווח.

## קרא קודם, בסדר הזה

1. `docs/agent/entry-point.md` — כללי עבודה מחייבים (כולל שרשרת הקריאה שהוא מפנה אליה).
2. `docs/research/global-search-spec.md` ★★ — האפיון. זה המסמך המחייב של המשימה.
3. `docs/domains/search.md` — החוזה הנוכחי שהולך להשתנות (ישתכתב כחלק מהעבודה).
4. ה-issues ב-Linear (פרויקט YM Tax CRM): **MAT-92, MAT-93, MAT-94, MAT-95** —
   קריטריוני הקבלה שם נגזרו מהאפיון, סעיף §12.

## היקף

| Issue | בלוק באפיון | תלות |
|---|---|---|
| MAT-92 | B1 — בקאנד: חוזה `matches` + `SearchMatchRepository` (UNION ALL) + פרסר מונח | אין |
| MAT-93 | B2 — מיגרציית btree לעמודות התאמה מדויקת | מקביל ל-92 |
| MAT-94 | F1 — עמוד `/search`: תוצאות-בלבד, שתי רשימות, מחיקת פילטרים ומצב לקוח | חסום ע"י 92 |
| MAT-95 | F2 — טאב קלסרים בעמוד הלקוח (פרונט בלבד, ה-endpoint קיים) | עצמאי |

**מחוץ להיקף:** MAT-96 (שלב 2 — תחילית/הכלה) gated על מדידה; אל תכין לו תשתית מעבר
לעמודת ה-rank שהאפיון מגדיר.

## עובדות מפתח מהאפיון — לא לגלות מחדש

- הפיד (dossier) **נמחק**, לא עובר. `items`/`SearchItemGroups`/`SearchItemRepository` נמחקים.
- קליק על לקוח ב-`/search` **מנווט** ל-`/clients/:id`. אין ניווט אוטומטי בשום תרחיש (D1).
- מונח מספרי מותאם רק מול מזהים גלויים למשתמש (D4). תקופה מנורמלת: `03/2026`↔`2026-03` (D5).
- `SearchMatch` נושא זהות לקוח (`client_record_id`+`client_name`+`client_office_number`) —
  מודל חדש, לא השמנה של `SearchItem`.
- `GET /search`: פרמטר `search` בלבד (+דפדוף); שבעה פרמטרים נמחקים.
  `GET /search/items`: `search`+`result_type`, בלי `client_record_id`.
- מחיקת חלקים מ-MAT-63/64/87 = חוב שנסגר במודע. לתעד, לא לשחזר.
- שום trigram/GIN בשלב 1 (D7) — btree בלבד.
- לא משתנים: `require_role(ADVISOR, SECRETARY)`, סינון soft-delete פר-דומיין,
  `entity_route` כבעלים יחיד של URLים.

## אילוצים ומלכודות

- שלושה repos נפרדים (`backend`, `frontend`, `docs`); השורש אינו repo — `git -C <subdir>`.
- ADR-0001/0003: אין legacy, אין aliases, אין fallbacks. מחיקה = מחיקה.
- `docs/domains/search.md` + `docs/architecture/api-contracts.md` מתעדכנים **באותה משימה**
  עם שינוי החוזה (MAT-92), לא בסוף.
- שינוי חוזה: לחדש `backend/openapi.json` ואז `npm run gen:types` (דורס את `generated.ts`
  במקום — לבודד diff לפני).
- פרונט: `npm run typecheck` (לא `npx tsc --noEmit` — no-op שקט). אין jsdom/RTL —
  בדיקות = פונקציות טהורות + SSR markup בלבד.
- בקאנד: `.venv` + `APP_ENV=test JWT_SECRET=x` להרצת pytest. Python 3.13 (Render).
- מיגרציות: downgrade מוריד בדיוק את מה ש-upgrade הוסיף.
- לפני שמסמנים AC כעשוי — להריץ את האימות המתאים מ-`docs/workflow/verification.md`.

## תוצר מבוקש מהשיחה

1. תוכנית ביצוע: סדר בין ה-issues, פירוק לקומיטים לכל repo, אילו בדיקות קיימות
   נשברות בכוונה (רשימה), נקודות סיכון.
2. אחרי אישור — מימוש לפי התוכנית, issue אחר issue, עם אימות מלא בכל שלב.
