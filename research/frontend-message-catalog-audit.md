## Scope

This file owns only:

- A point-in-time audit of frontend message catalogs.
- Findings about duplicated copy, inconsistent wording, typography, and unnecessary local copies of global messages.

This file must not contain:

- Permanent frontend architecture or copywriting rules.
- Product behavior changes.
- An implementation checklist that is treated as a source of truth.

Source of truth: research only

# Frontend Message Catalog Audit

Audit date: 2026-07-27

## Scope and method

Reviewed all 56 files matching:

- `frontend/src/messages.ts`
- `frontend/src/features/*/messages.ts`
- `frontend/src/features/*/errorMessages.ts`

The catalogs contain 1,983 static string literals. Exact-text grouping found 208 repeated texts across
575 occurrences. These raw counts include legitimate repetition: a short domain label such as
`באיחור`, `שולם`, or `תאריך יעד` can correctly appear in several feature-owned catalogs. This report
therefore lists only actionable or review-worthy findings and does not propose moving every repeated
domain term into the global catalog.

The audit did not review arbitrary inline JSX strings outside the message catalogs.

## Summary

| Priority | Finding | Scope |
|---|---|---:|
| High | Mixed Hebrew/ASCII punctuation in the same terms | 4 recurring term families |
| High | Feature-local copies of values that already exist globally | 9 occurrences |
| High | Same VAT error/action has multiple forms inside the VAT feature | 5 groups |
| Medium | Same feature repeats identical copy under several keys | 15 clear groups |
| Medium | Cross-feature generic success/error messages are duplicated | 11 clear groups |
| Medium | Loading, action, and empty-state wording is not consistently patterned | 6 families |
| Low | One trailing whitespace defect | 1 occurrence |

## 1. Typography that should be normalized

These are not meaningful copy variants. They are typographic drift and should use one spelling.

| Term | Current forms | Evidence | Recommendation |
|---|---|---|---|
| VAT | `מע"מ` (46 occurrences), `מע״מ` (56 occurrences) | Concentrated in `annualReports`, `clients`, `vatReports`, `dashboard`, and `binders` catalogs | Use `מע״מ` consistently |
| Client/charge number abbreviation | `מס'` and `מס׳` | `annualReports/messages.ts:124`, `clients/messages.ts:17`, `advancedPayments/messages.ts:157`, `taxCalendar/messages.ts:55` | Use `מס׳` consistently |
| Total abbreviation | `סה"כ` (11), `סה״כ` (7) | `annualReports/messages.ts:31`, `vatReports/messages.ts:124`, `advancedPayments/messages.ts:330` | Use `סה״כ` consistently |
| “by” abbreviation | `ע"י` (2), `ע״י` (1) | `audit/messages.ts:9`, `vatReports/messages.ts:171`, `binders/messages.ts:111` | Use `ע״י` consistently |

Specific same-feature conflicts:

- `clients/messages.ts` uses both `תדירות דיווח מע"מ` (`:70`) and
  `תדירות דיווח מע״מ` (`:201`, `:265`, `:289`).
- `clients/messages.ts` uses both `תקרת פטור מע"מ` (`:71`) and
  `תקרת פטור מע״מ` (`:204`, `:269`).
- `vatReports/messages.ts` uses both `מע"מ עסקאות` (`:103`) and
  `מע״מ עסקאות` (`:84`, `:119`), plus the same split for input VAT.
- `vatReports/errorMessages.ts` uses both `תיק מע"מ` (`:14`, `:20`, `:24`) and
  `תיק המע״מ` (`:16`) for the same entity.
- All loading and placeholder ellipses currently use three ASCII dots (`...`, 63 occurrences).
  This is internally consistent; changing them to `…` would be cosmetic and is not recommended
  unless a project-wide typography rule is adopted.

## 2. Local copies that already exist globally

The following feature keys duplicate `GLOBAL_UI_MESSAGES` exactly. Consumers should import and use
the global value directly. Do not replace these with feature aliases that merely point to the global
key.

| Feature key | Existing global key |
|---|---|
| `ADVANCED_PAYMENTS_MESSAGES.batchColumns.clientNameHeader` | `GLOBAL_UI_MESSAGES.common.clientName` |
| `ADVANCED_PAYMENTS_MESSAGES.overviewSort.clientNameLabel` | `GLOBAL_UI_MESSAGES.common.clientName` |
| `AUTH_MESSAGES.login.featurePills` item `לקוחות` | `GLOBAL_UI_MESSAGES.common.clients` |
| `CLIENTS_MESSAGES.columns.fullName` | `GLOBAL_UI_MESSAGES.common.clientName` |
| `CLIENTS_MESSAGES.createIdentity.clientNameField` | `GLOBAL_UI_MESSAGES.common.clientName` |
| `NOTES_MESSAGES.clientTab.title` | `GLOBAL_UI_MESSAGES.common.notes` |
| `SEARCH_MESSAGES.hints` item label `לקוחות` | `GLOBAL_UI_MESSAGES.common.clients` |
| `TASKS_MESSAGES.details.status` | `GLOBAL_UI_MESSAGES.common.status` |
| `TASKS_MESSAGES.details.client` | `GLOBAL_UI_MESSAGES.common.client` |

Array entries such as `featurePills` and structured hint objects do not need a new exported alias.
They can use the global value directly inside the structure if retaining the structure is useful.

## 3. Same-feature duplicates that should use one owned key

These pairs or groups have the same wording and the same semantic role inside one feature. Keep one
feature-owned key and update the feature's consumers to use it directly.

| Feature | Duplicate keys | Text |
|---|---|---|
| Advanced payments | `editableSections.notesPlaceholder`, `createModal.notesPlaceholder` | `הערות...` |
| Advanced payments | `detail.turnoverMissingBadge`, `batchColumns.missingTurnoverBadge` | `חסר מחזור` |
| Advanced payments | `turnoverRefresh.mismatchBadge`, `batchRow.vatMismatchLabel` | `אי-התאמת מע״מ` |
| Advanced payments | `detail.periodTurnoverLabel`, `editableSections.periodTurnoverLabel` | `מחזור לתקופה` |
| Advanced payments | `readonlySections.calculatedAmountLabel`, `editableSections.calculatedAmountLabel`, `createModal.calculatedAmountLabel` | `סכום מחושב` |
| Advanced payments | `editableSections.overrideAmountLabel`, `createModal.overrideAmountLabel` | `סכום עקיפה (אופציונלי)` |
| Advanced payments | `page.createYearlySchedule`, `clientHeader.createYearlySchedule` | `צור לוח שנתי` |
| Advanced payments | `clientTab.title`, `clientTab.paginationLabel` | `מקדמות` |
| Advanced payments | `clientStats.totalExpectedTitle`, `detail.annualExpectedLabel` | `סה״כ צפוי` |
| Advanced payments | `clientStats.totalPaidTitle`, `detail.annualPaidLabel` | `סה״כ שולם` |
| Annual reports | `reportHistoryTable.taxDueHeader`, `yearComparisonModal.taxDueLabel` | `חבות מס` |
| Annual reports | `reportHistoryTable.refundDueHeader`, `yearComparisonModal.refundDueLabel` | `החזר מס` |
| Clients | `statusCard.noBusinesses`, `businessesCard.empty` | `אין עסקים רשומים` |
| Notifications | `tab.loading`, `page.loading` | `טוען הודעות...` |
| VAT reports | `breakdown.invoiceVat`, `categoryTable.invoiceVat` | `מע"מ בחשבוניות` |

Do not consolidate keys whose identical text serves a different semantic role. Examples that should
remain separate include a table heading versus a status value (`שולם`), an ARIA label versus a
visible action, and a singular entity label versus a pagination label.

## 4. VAT feature wording conflicts

The VAT catalogs contain several cases where the same operation or entity has unnecessary wording
variants:

| Concern | Current variants | Evidence | Recommended canonical form |
|---|---|---|---|
| Create work item failure | `שגיאה ביצירת תיק מע"מ`, `שגיאה ביצירת תיק המע״מ` | `vatReports/errorMessages.ts:16`, `:24`; also `dashboard/errorMessages.ts:6` | `שגיאה ביצירת תיק מע״מ` |
| File report action | `הגש`, `הגש מע"מ` | `vatReports/messages.ts:12-13` | Keep only the context-appropriate key; where the button is already inside a VAT screen, prefer `הגש` |
| Open/create entity | `פתיחת דוח מע״מ`, `דוח מע״מ חדש`, `פתיחת תיק מע"מ חדש` | `vatReports/messages.ts:8-9`, `:42` | Choose whether the user creates a “report” or a “case” and use that product term consistently |
| Empty entity wording | `אין תיקי מע"מ`, `אין תיקים בתקופה זו`, `לא נמצאו תיקים...` | `vatReports/messages.ts:134-136` | Keep contextual empty-state differences, but normalize the entity spelling to `תיקי מע״מ` |
| Final override amount | `סכום מע"מ סופי (עוקף)`, `סכום מע"מ עוקף` | `vatReports/messages.ts:70`, `:210` | Prefer `סכום מע״מ סופי (עוקף)` if both refer to the same value |

The “report” versus “case” choice affects product terminology, so it should be confirmed before
mechanically changing keys. The punctuation changes do not need product review.

## 5. Cross-feature generic duplicates

These messages are not domain copy. They are repeated for the same generic operation and are
reasonable candidates for a single global key only when all listed consumers mean exactly the same
thing.

| Text | Occurrences |
|---|---|
| `הפעולה בוצעה בהצלחה` | `vatReports/messages.ts:179`, `workQueue/messages.ts:66` |
| `המשימה נוצרה בהצלחה` | `tasks/messages.ts:39`, `workQueue/messages.ts:67` |
| `המשימה עודכנה בהצלחה` | `tasks/messages.ts:40`, `workQueue/messages.ts:68` |
| `יצירת המשימה נכשלה` | `tasks/errorMessages.ts:18`, `workQueue/errorMessages.ts:14` |
| `הסיסמאות אינן תואמות` | `auth/errorMessages.ts:10`, `users/errorMessages.ts:10` |
| `שגיאה ביצירת חיוב` | `charges/errorMessages.ts:6`, `clients/errorMessages.ts:7`, `dashboard/errorMessages.ts:7` |
| `שגיאה ביצירת לקוח` | `clients/errorMessages.ts:10`, `dashboard/errorMessages.ts:4` |
| `שגיאה בשחזור לקוח` | `clients/errorMessages.ts:9`, `dashboard/errorMessages.ts:5` |
| `שגיאה ביצירת מקדמה` | `advancedPayments/errorMessages.ts:6`, `dashboard/errorMessages.ts:3` |
| `שגיאה בטעינת הדוח` | `annualReports/errorMessages.ts:2`, `reports/errorMessages.ts:3` |
| `פורמט תקופה חייב להיות YYYY-MM` | `charges/messages.ts:123`, `vatReports/errorMessages.ts:8` |

Ownership should follow semantics:

- Task create/update messages belong to the tasks feature and should be imported from its public
  surface by work queue.
- Client, charge, advance-payment, and VAT creation errors belong to their domain feature; dashboard
  should consume the owning feature export rather than keep a copy.
- Password validation shared by auth and user administration may justify a global validation message
  only if both flows intentionally share the same password policy.
- Do not create a second “shared errors” catalog solely to collect two strings.

## 6. Wording that needs a single convention

### Error construction

Error catalogs mix:

- `שגיאה ב...` — for example `שגיאה בטעינת דוח`.
- `<operation> נכשל/נכשלה` — for example `טעינת העבודה לטיפול נכשלה`.
- `לא ניתן...` — for example `לא ניתן לזהות תפקיד משתמש`.

The third form is useful when it describes a user-facing limitation. The first two are otherwise
interchangeable and create avoidable drift. Prefer one pattern for technical operation failures;
`<operation> נכשלה` is shorter, while `שגיאה ב...` is currently more common. This is a copy decision,
not an architecture change.

Concrete duplicate-intent examples:

- `workQueue/errorMessages.ts:4-5`: `שגיאה בטעינת תור העבודה` versus
  `טעינת העבודה לטיפול נכשלה` for the same load failure.
- `advancedPayments/errorMessages.ts:6-10`: CRUD failures use `שגיאה ב...`, while adjacent bulk
  failures use `... נכשל`.
- `vatReports/errorMessages.ts:14-24`: load/create failures use several noun and punctuation forms.

### Loading text

Loading messages alternate between `טוען X...`, `טוען את X...`, and present-tense plural such as
`בודקים...` / `מתחברים...`. Person agreement can be intentional in auth flows, but entity-loading
copy should consistently use `טוען <entity>...` unless the article is needed for natural Hebrew.

Clear candidates:

- `clients/messages.ts:46` — `טוען פרטי לקוח...`
- `clients/messages.ts:171` — `טוען את פרטי הלקוח...`

These refer to the same operation and should use one form.

### Action labels

The global catalog defines `back: חזרה`, while feature catalogs also use `חזור`
(`charges/messages.ts:48`, `signatureRequests/messages.ts:14`). For the same back-button behavior,
use the global action directly. Do not add a global `backAction` alias.

The global catalog defines `edit: עריכה`, while visible actions vary between `עריכה`, `עריכת ...`,
and context-specific verbs. This is not automatically wrong: `עריכת לקוח` is a useful explicit menu
label. Only generic buttons should use the global `עריכה`.

### Required-field markers

Some message values embed `*` (`לקוח *`, `שנת מס *`, `שם מלא *`) while related fields use the plain
label. If the form components already render required state, remove the star from message values and
let the component own the marker. If they do not, retain the variants; this cannot be normalized
safely from catalog inspection alone.

### Empty states

Generic empty states should use `GLOBAL_UI_MESSAGES.common.noData` or `.noResults` only when no
domain explanation is needed. Domain-specific empty states such as `אין מקדמות לשנה 2026` or
`אין חשבוניות בקטגוריה זו` should remain feature-owned.

## 7. Defect

`frontend/src/features/binders/messages.ts:65` contains a trailing space:

```ts
officeClientNumber: "מס' לקוח "
```

It should match the canonical spelling without trailing whitespace: `מס׳ לקוח`.

## 8. Repetitions that should not be over-consolidated

The largest exact-repeat groups are mostly domain/status vocabulary:

| Text | Occurrences | Decision |
|---|---:|---|
| `באיחור` | 12 | Keep with the owning status maps; do not create a universal status alias |
| `ת.ז / ח.פ` | 12 | Consider global only if the same display convention is intentionally universal |
| `דוח שנתי` | 9 | Keep domain-owned where it identifies an entity |
| `שולם` | 9 | Keep context-owned; label, column, and status semantics differ |
| `מסמכים` | 8 | Keep context-owned unless used as a generic navigation label |
| `תקופה` | 8 | Too generic to centralize safely |
| `סוג` | 7 | Too generic to centralize safely |
| `מקדמות` | 7 | Keep advanced-payments ownership; cross-feature consumers may import its public label if needed |

Creating global constants for every short repeated noun would increase indirection without reducing
semantic duplication. Consolidation should be limited to:

1. Values already owned by `GLOBAL_UI_MESSAGES`.
2. Exact same-feature duplicates with the same role.
3. Cross-feature consumers of a message clearly owned by one domain feature.

## Recommended implementation order

1. Normalize punctuation and remove the trailing space.
2. Remove the 9 local copies of existing global values; update consumers to import
   `GLOBAL_UI_MESSAGES` directly.
3. Collapse the 15 clear same-feature duplicate groups by updating consumers to one existing key.
4. Replace dashboard/work-queue copies with direct imports from the owning feature's public export.
5. Review the VAT “report” versus “case” terminology before changing those product-facing nouns.
6. Decide one error-copy pattern; apply it in a separate, mechanical pass to avoid mixing product
   terminology changes with cleanup.

No alias, wrapper, compatibility key, or new generic message layer is needed for these changes.
