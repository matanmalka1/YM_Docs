> **Research reference only. Not tax advice and not product source of truth.**
> This document contains year-sensitive external tax research. Re-verify every legal/tax fact against official sources before using it for product behavior.

# Israeli Annual Income Tax Report (דוח שנתי) — Source-Grounded Research for Software Modeling

**Prepared for:** Annual Reports domain rebuild (CRM for Israeli accounting/tax firms)
**Retrieval date for all sources below:** 2026-06-18
**Scope note:** This is research to support *software modeling*, not tax advice. Every legal claim must be validated by a licensed Israeli רו"ח / יועץ מס before it drives any real filing. See Section I.

**How to read the tags**
- `[FACT — <source>]` = sourced to a page/PDF actually fetched or to a verified search result, with the tier noted.
- `[INFERENCE]` = my own modeling interpretation, not a legal claim.
- `UNVERIFIED — NOT FOUND` = I looked and could not confirm against an acceptable source in this pass.

**Fetch reliability caveat for this pass.** The Tax Authority HTML *service pages* on `gov.il` returned bot-detection blocks on direct fetch. I was, however, able to fetch the **official form PDF for טופס 1301 (tax year 2025)** directly from the `gov.il` blob store, and to confirm the existence/identity of the other official service pages via search results. Where a fact rests only on a confirmed-existing official page I could not fully render, or on a secondary source, it is flagged. This is the single biggest limitation of this pass and is itemized in Section H.

---

## Phase 0 — Source map (official entry points used)

| # | Entry point | Body | Tier | URL | Fetched? |
|---|---|---|---|---|---|
| 1 | טופס 1301 — official annual return PDF, TY2025 | רשות המסים | 1 | gov.il blob (see Source Index) | ✅ Full text |
| 2 | Annual report service page TY2025 (1301 online) | רשות המסים | 1 | gov.il/he/service/reporting-and-payment-2025-annual-tax-report-for-individuals | ⚠️ Blocked; confirmed via links/snippets |
| 3 | טופס 6111 service page | רשות המסים | 1 | gov.il/he/service/itc6111 | ⚠️ Confirmed via snippet |
| 4 | טופס 106 service page | רשות המסים | 1 | gov.il/he/service/itc-106 | ⚠️ Confirmed via snippet |
| 5 | טופס 806/857 service page | רשות המסים | 1 | gov.il/he/service/itc806 | ⚠️ Confirmed via snippet |
| 6 | הצהרת הון / טופס 1219 service page | רשות המסים | 1 | gov.il/he/service/itc1219 | ⚠️ Confirmed via snippet |
| 7 | Refund-without-obligation guide (attachments, fields) | רשות המסים | 1 | gov.il/he/pages/tax-return-application-guide | ⚠️ Confirmed via snippet |
| 8 | הוראת ביצוע 3/2018 (amended returns) | רשות המסים | 1 | gov.il blob (PDF) | ⚠️ Linked from כל זכות; not separately fetched |
| 9 | הגשת דו"ח שנתי למס הכנסה (process hub) | כל זכות | 2 | kolzchut.org.il | ✅ Full text |
| 10 | תקנות מס הכנסה (פטור מהגשת דין וחשבון), תשמ"ח-1988 | משרד האוצר (text) | 1/2 | he.wikisource.org | ⚠️ Via snippet (Wikisource mirror) |
| 11 | פקודת מס הכנסה — סעיפים 131, 132 (text) | המחוקק | 1/2 | Wikisource / hilan mirror | ⚠️ Via snippet only |
| 12 | נבו — מדור טפסים (form-number registry) | נבו | 1 | nevo.co.il | ⚠️ Via snippet |

> **Authority note:** Wikisource and commercial legal mirrors (hilan, nevo) are being used here *as a text witness to primary law*. Treat them as Tier 1 *content* with Tier 2 *custody*. The authoritative text is the official פקודה on the legislative database; re-verify section text there before relying on exact wording.

---

## A. Source index table

| Topic | Source title | Issuing body | Tier | URL | Relevant section / field | Tax year(s) | Reliability note |
|---|---|---|---|---|---|---|---|
| Annual return form | טופס 1301 — דוח ליחיד (PDF) | רשות המסים, אגף טכנולוגיות דיגיטליות ומידע | 1 | gov.il/BlobFolder/service/reporting-and-payment-2025-annual-tax-report-for-individuals/he/Service_Pages_Income_tax_annual-report-2026_itc1301-2025.pdf | Whole form; "מעודכן ל-1.2026" | TY2025 | **Fetched in full.** Highest-reliability item in this report. |
| Filing obligation | סעיף 131 לפקודת מס הכנסה | המחוקק | 1 (via mirror) | he.wikisource.org (פקודת מס הכנסה #סעיף 131) | §131(a) list of filers | current | Text via Wikisource/aggregators; not fetched from official DB |
| Filing deadline | סעיף 132 לפקודת מס הכנסה | המחוקק | 1 (via mirror) | hilan.co.il mirror of פקודה ch. ח' | §132(a),(b) | current | Verbatim statute via commercial mirror only |
| Exemption from filing | תקנות מס הכנסה (פטור מהגשת דין וחשבון), תשמ"ח-1988 | משרד האוצר | 1 (via mirror) | he.wikisource.org | regs 1–5; thresholds | current | Wikisource mirror |
| Process / deadlines / amendments | הגשת דו"ח שנתי למס הכנסה (הליך) | כל זכות | 2 | kolzchut.org.il/he/הגשת_דו"ח_שנתי_למס_הכנסה | full page | TY2025 dates; updated 2026-03-11 | Plain-language; cites official law and Authority |
| 6111 form | נספח לדו"ח השנתי - טופס 6111 | רשות המסים | 1 | gov.il/he/service/itc6111 | online-only filing | current | Page confirmed; not rendered |
| 6111 obligation threshold | טופס 1301 (printed trigger) | רשות המסים | 1 | (1301 PDF, part א) | "מחזור … מעל 256,410 ₪ (ללא מע"מ)" | TY2025 | From fetched form |
| 6111 threshold (incl. VAT) | דע את זכויותיך; gov.il annual-tax-report-2020 | רשות המסים | 1 | gov.il/he/service/annual-tax-report-2020 | 300,000 ₪ כולל מע"מ | TY2019–2020 stated | Year-sensitive; reconciles with ex-VAT figure |
| Salary certificate | אישור על משכורת וניכוי מס — טופס 106 | רשות המסים | 1 | gov.il/he/service/itc-106 | employer annual cert | current | Page confirmed via snippet |
| Capital-market withholding cert | טופס 867 (+867א רווח הון מני"ע) | רשות המסים / נבו registry | 1 | nevo.co.il מדור טפסים; bank pages | "867 א-ה" referenced | current | 867א existence confirmed (nevo) |
| Other-payer withholding cert | אישור שנתי על ניכוי במקור — טופס 806/857 | רשות המסים | 1 | gov.il/he/service/itc806 | תקנות ניכוי במקור התשמ"א-1980 | current | Page confirmed |
| Capital statement | הצהרת הון — טופס 1219 | רשות המסים | 1 | gov.il/he/service/itc1219 | demand-based filing | current (online since 2025) | Page confirmed |
| Capital statement legal basis | סעיף 135 לפקודה | המחוקק | 1 (via Tier 3 citation) | cited by multiple CPA firms | §135(1) / §135(1)(a) | current | Section cited only by Tier 3; verify clause |
| Attachments / annex checklist | מדריך החזר מס; דע את זכויותיך | רשות המסים | 1 | gov.il/he/pages/tax-return-application-guide | annex list, field mapping | TY2024+ | Confirmed via snippet |
| Amended return procedure | הוראת ביצוע מס הכנסה 3/2018 | רשות המסים | 1 | gov.il blob (Policy_IncomeTaxInst_hor_3-2018) | whole instruction | 2018+ | Linked, not separately fetched |
| Surtax (מס יסף) | סעיף 121ב לפקודה | המחוקק | 1 | (1301 PDF field) | §121ב(ה) | TY2025 | Threshold from fetched form |

---

## B. Per-form specifications

### B.1 טופס 1301 — דוח ליחיד (Annual return for an individual) — **TY2025, fully verified**

- **Official name:** טופס 1301 — "דין וחשבון על ההכנסות בארץ ובחו"ל בשנת המס 2025" (return on income in Israel and abroad for the tax year). `[FACT — gov.il 1301 PDF, Tier 1]`
- **Version stamp:** "מעודכן ל-1.2026", issued by ר"י, אגף בכיר טכנולוגיות דיגיטליות ומידע. The form covers the year 1.1.2025–31.12.2025. `[FACT — 1301 PDF]`
- **Purpose:** the individual's primary annual income tax return (income from all sources, Israel + abroad, for the filer, spouse and children under 18). `[FACT — §131; 1301 PDF]`
- **When required:** when filing is obligatory under סעיף 131, or voluntarily to claim a refund ("אני מגיש דוח … למרות שאיני חייב — בקשה להחזר מס"). `[FACT — 1301 PDF, part א]`
- **Structure (verified parts/letters):** The form runs 4 pages and is organized into lettered parts:
  - **א.** פרטים כלליים (general details, family status, bank account for refunds, 6111 flag, numerous yes/no declarations).
  - **ב.** פרטים אישיים (personal/business identifiers, email/SMS notification consent).
  - **ג.** הכנסות מיגיעה אישית החייבות בשיעורי מס רגילים (personal-exertion income at regular rates).
  - **ד.** הכנסות חייבות בשיעורי מס רגילים (regular-rate income).
  - **ה.** הכנסות חייבות בשיעורי מס מיוחדים (special-rate income — interest, dividends, rental tracks, lottery, etc.).
  - **ו.** מוסד כספי (financial institution).
  - **ז.** נתונים נוספים (turnover, loss offsets, §9(5) exempt income).
  - **ח.** רווח הון ושבח מקרקעין (capital gains / land appreciation).
  - **ט.** הכנסות חו"ל (foreign income; cross-refs נספח ד').
  - **י.** הכנסות/רווחים פטורים (exempt income).
  - **יא.** פרטים נוספים ויתרות להעברה (carry-forwards to TY2026).
  - **יב.** ניכויים אישיים (personal deductions — pension, keren hishtalmut, disability-of-earning-capacity insurance, etc.).
  - **יג.** נקודות זיכוי (credit points — residency, children, single parent, new immigrant, discharged soldier, academic degree, etc.).
  - **יד.** זיכויים אישיים (personal credits — life insurance, pension as employee/self-employed, donations §46, etc.).
  - **טו.** מחזור למקדמות, ניכויים במקור, מס שבח (turnover for advances, withholding, land-appreciation tax). `[FACT — 1301 PDF]`
- **Verified key field codes (sample, for mapping):** salary/pension withholding = field **042** (from טופס 106); interest withholding = **043**; other withholding = **040**; donations to Israeli institutions = **037/237** (§46); rental at 10% = part ה line 24 (§122); foreign rental at 15% = line 25 (§122א); §9(5) exempt = field **109**. `[FACT — 1301 PDF]`
- **Verified declared signatures/sections referenced on the form:** §131 (officer may treat an improperly completed/unaccompanied return as not filed); §143 (paid preparer declaration); §144 ("בן הזוג הרשום" power-of-attorney signing); §224 (preparer's responsibility). `[FACT — 1301 PDF]`
- **Notes/uncertainties:** Many checkboxes pull in dependent forms (see B.6). Field code stability across years is not guaranteed — re-map per tax-year form. `[INFERENCE]`

### B.2 טופס 6111 — נספח דוח התאמה למס / financial-statement annex

- **Official name:** "נספח לדו"ח השנתי — טופס 6111" (also "נספח לטופס הדו"ח השנתי ליחיד ולחבר בני אדם"). `[FACT — gov.il/he/service/itc6111, Tier 1, confirmed]`
- **Purpose:** structured, coded restatement of the financial statements — profit & loss, balance sheet, and tax-adjustment (דוח התאמה למס) — attached to the annual return. It does **not** replace audited financial statements; it is additional. `[FACT — official "דברי הסבר"/דע את זכויותיך, Tier 1 content]`
- **When required:** business/self-employed (and companies) above a turnover threshold. The **1301 form's printed trigger** is turnover **over 256,410 ₪ excluding VAT** (TY2025) → checkbox "חייב בטופס 6111". `[FACT — 1301 PDF]` The Authority's guide states the threshold as **300,000 ₪ including VAT**. `[FACT — gov.il annual-tax-report-2020, Tier 1]` These reconcile (ex-VAT × ~1.17 ≈ incl-VAT). **Year-sensitive — re-verify the exact figure per year.**
- **Exemptions (from the official "דברי הסבר"):** entities under §3(ז), contractors under §8א, banks, insurance companies, and farmers (תוספת י"ב). `[FACT — official explanatory note, surfaced via Tier 3 mirror — verify on gov.il]`
- **Filing channel:** online only (שידור), per §240ב(ג); after entry, the data-summary page with barcode is printed, signed, and submitted with the return. `[FACT — official guide; §240ב(ג) cited by Tier 3 — verify section]`
- **Structure:** three parts — P&L, tax adjustment, balance sheet. `[FACT — multiple, incl. official guide content]`

### B.3 טופס 106 — אישור על משכורת וניכוי מס (employer annual certificate)

- **Official name/purpose:** annual employer certificate of salary paid and tax withheld; also shows work months, employer/employee pension contributions, and credit points. `[FACT — gov.il/he/service/itc-106, Tier 1, confirmed]`
- **When required as attachment:** an employee, or recipient of a pension from a former employer / provident fund, attaches טופס 106 to the return. In the return, salary-withholding is copied to field **042**, and the 106 section number equals the return field **172/158**. `[FACT — gov.il refund guide, Tier 1]`
- **Notes:** issued by each employer/payer; multiple 106s if multiple employers (relevant to excess-contribution form 134). `[FACT — refund guide]`

### B.4 טופס 867 (and 867א–ה) — capital-market withholding certificate

- **Official name/purpose:** annual certificate from banks / stock-exchange members summarizing gains/losses from deposits, securities, dividends and interest, and the tax withheld. **867א** specifically = "אישור ניכוי מס במקור על רווח הון מניירות ערך". `[FACT — nevo מדור טפסים registry, Tier 1; bank pages Tier 3]`
- **When required as attachment:** withholding certificates from capital-market payers (טופס 867, 867 א-ה) are attached to the return; interest-withholding from these flows to return fields **078/126/142**. `[FACT — official annex checklist + refund guide, Tier 1 content]`
- **Notes:** only **realized** gains/losses appear; needed to assess refunds where in-year withholding ignored prior-year losses. `[FACT — finupp, Tier 3 — corroborative only]`
- **Uncertainty:** the full set "867א through 867ה" is referenced by the official annex checklist, but I did not enumerate each sub-letter's exact title against an official page. `UNVERIFIED — NOT FOUND` (sub-letters beyond 867א).

### B.5 טופס 806 / 857 — annual withholding certificate from other payers

- **Official name/purpose:** "אישור שנתי על ניכוי מס הכנסה … מתשלומים המחייבים ניכוי מס במקור", per תקנות מס הכנסה (אישור בדבר ניכוי במקור), התשמ"א-1980. Issued by any payer obliged to withhold; lists payments, VAT and tax withheld. `[FACT — gov.il/he/service/itc806, Tier 1, confirmed]`
- **When required:** withholding (other than from salary) is allowed only on the basis of original annual certificates; a representative (רו"ח/יועץ מס) may instead attach the representative's own certification with a detailed schedule. `[FACT — official annex checklist, Tier 1 content]`

### B.6 הצהרת הון — טופס 1219 (capital statement) — **related but separate filing**

- **Official name/purpose:** declaration of all assets and liabilities of the taxpayer, spouse and children under 18, valued at a specific date set by the assessing officer; used to compare two statements over time and test unexplained capital growth (הפרשי הון). `[FACT — gov.il/he/service/itc1219, Tier 1, confirmed]`
- **Legal basis:** סעיף 135 לפקודה (officer's power to demand a detailed statement of capital/assets); commonly cited as §135(1)/(1)(a). `[FACT — §135 cited by multiple CPA firms, Tier 3 — verify exact clause on official text]`
- **Relationship to the annual report (important for the data model):**
  - It is **NOT** part of the annual return and is **not** self-initiated. It is filed **only on the assessing officer's demand**, typically on opening a business (first statement) and then every ~3–5 years. `[FACT — multiple Tier 3 CPA sources; consistent]`
  - Deadline: within **120 days** of the demand (or of the declaration date, whichever is later). `[FACT — Tier 3 consistent]`
  - For בעלי שליטה (controlling shareholders) it may in some cases be submitted together with the annual returns. `[FACT — Tier 3 — flag]`
  - Online filing available since 2025 (still optional vs. manual as of 2026). `[FACT — Tier 3 consistent]`
- **Modeling implication:** model הצהרת הון as a *separate, demand-triggered obligation entity* linked to the client/tax-file, **not** as a child of the annual-report entity. `[INFERENCE]`

### B.7 Common annexes (נספחים) — verified numbers

| Annex / form | Title (gloss) | Source |
|---|---|---|
| נספח א' = טופס 1320 | חישוב ההכנסה החייבת מעסק או ממשלח יד (business/profession taxable income; sufficient where no double-entry) | `[FACT — official annex checklist, Tier 1 content]` |
| נספח ב' = טופס 1321 | additional income from property (rental, key money, interest, dividend, authors'/lecturers' fees, casual transactions) | `[FACT — official annex checklist]` |
| נספח ג' = טופס 1322 | רווח הון מניירות ערך סחירים (capital gains from traded securities); ג1, ג2 referenced in 1301 field 256 | `[FACT — official annex checklist + 1301 PDF]` |
| נספח ד' | הכנסות חו"ל (foreign income); 1301 field 290/40 | `[FACT — 1301 PDF]` |
| טופס 1399 / 1399י | רווח הון ממכירת נכסים שאינם ני"ע סחירים (capital gains, non-traded assets) | `[FACT — official guide content; ynet Tier 3 corroborates 1399י]` |
| טופס 134 | חישוב הכנסה בגין תשלומים עודפים של מעביד לקרן השתלמות/קופ"ג + אובדן כושר עבודה | `[FACT — 1301 PDF + refund guide]` |
| טופס 150 | אחזקה בחבר בני אדם תושב חוץ (foreign company holding) | `[FACT — 1301 PDF]` |
| טופס 1504 | דיווח על שותפות (partnership) | `[FACT — 1301 PDF]` |
| טופס 119 | זיכוי בעד סיום תואר אקדמי / לימודי מקצוע (academic-degree credit) | `[FACT — 1301 PDF]` |
| טופס 1343 | ניכוי נוסף בשל פחת (additional depreciation) | `[FACT — nevo registry, Tier 1]` |
| טופס 4435 | הצהרה בדבר קביעת "בן זוג רשום" (registered-spouse declaration) | `[FACT — nevo registry + kolzchut]` |
| טופס 1385 | עסקאות עם צדדים קשורים בחו"ל §85א (related-party / transfer pricing) | `[FACT — 1301 PDF]` |
| טופס 1345 / 1346 | חוות דעת §131ד / עמדה חייבת בדיווח §131ה (reportable opinion / position) | `[FACT — 1301 PDF]` |
| טופס 1348 / 1213 | תושב חוץ §131(5ה) / פעולה חייבת בדיווח §131(ז) | `[FACT — 1301 PDF]` |
| טופס 137 | דוח מקוצר ליחיד בעל עסק קטן (short return, small business) | `[FACT — 1301 PDF + kolzchut]` |

> **Depreciation form caveat:** an older Tier-3 source listed "טופס 1342" for depreciation; the official nevo registry lists **1343** as "ניכוי נוסף בשל פחת". Treat the *general* business-depreciation computation as living inside נספח א' (1320); flag 1342 vs 1343 for verification. `[INFERENCE / flag]`

---

## C. Lifecycle table

| Stage | Trigger event | Required data | Validation / check | Supporting source | Tag |
|---|---|---|---|---|---|
| Open work item | Filing obligation exists under §131, OR officer demand, OR voluntary refund claim | Client identity, tax file no., tax year, marital/registered-spouse status | Confirm obligation vs. exemption (תקנות הפטור) | §131; תקנות הפטור; kolzchut | FACT |
| Collect client materials | Year-end | 106s, 867/867א, 806/857, donation receipts (§46), pension/keren hishtalmut certificates, ביטוח לאומי certificates | Original annual certificates required for non-salary withholding | official annex checklist | FACT |
| Enter income | — | Per-category figures into 1301 parts ג–ה, ח, ט | Each income type to correct field/track; foreign income also on נספח ד' | 1301 PDF | FACT |
| Enter expenses / deductions | Business income present | נספח א' (1320) figures; deductions in part יב | If turnover > threshold → 6111 required & flagged in part א | 1301 PDF; itc6111 | FACT |
| Import / attach certificates | — | Map 106→042, interest→043, others→040; capital-market→078/126/142 | Representative may substitute own certification w/ schedule | refund guide | FACT |
| Calculate tax | All income/deductions/credits entered | Credits (part יג–יד), surtax §121ב if income > 721,560 (TY2025) | Surtax obligation forces filing even if PAYE withheld | 1301 PDF; bshcpa Tier 3 | FACT |
| Check missing data | Pre-submission | Completeness of attachments; declarations in part א | Officer may treat unaccompanied/incomplete return as **not filed** (§131) | 1301 PDF | FACT |
| Client approval / signature | — | "בן הזוג הרשום" signature (and spouse, or PoA per §144) | Single-spouse signature implies PoA declaration (§144) | 1301 PDF | FACT |
| Submit to authority | — | Online שידור (default) or manual in eligible cases | Manual: print 2 copies, submit, stamp one as receipt | kolzchut; gov.il service | FACT |
| Track post-submission | After submission | Submission confirmation (אישור הגשה); barcode for 6111/short return | A merely-transmitted (שודר) but not physically submitted return is **not** considered filed (manual track) | kolzchut | FACT |
| Store final PDF / confirmation | — | Final stamped return + confirmation; keep originals (esp. donations) | Officer may audit within 4 years of year of submission | 1301 PDF; kolzchut | FACT |
| Assessment / settle | הודעת שומה received | Pay balance (≤15 days of notice in 2020 guidance) or receive refund to registered bank account | Interest/linkage accrue from 1 Jan of following year | kolzchut; gov.il 2020 guide | FACT |

---

## D. Income categories table

| Category | Annual-report relevance | Supporting source documents | Form field / annex | Source / tag |
|---|---|---|---|---|
| Salary (משכורת) | Part ג line 3 (field 172/158) | טופס 106 (each employer) | 1301 part ג; field 042 for withholding | `[FACT — 1301 PDF + refund guide]` |
| Self-employed / business (עסק/משלח יד) | Part ג line 1 (field 150/170) | books; נספח א' (1320); 6111 if over threshold | 1301 part ג; נספח א'; 6111 | `[FACT — 1301 PDF]` |
| Rental — exempt track (פטור) | Exempt income reported in part י | tenancy records | 1301 field for exempt residential rent (part י line 42) | `[FACT — 1301 PDF]` |
| Rental — 10% track | Special-rate, part ה line 24 | rental records | §122; line 24 (10%) | `[FACT — 1301 PDF; §122 cited Tier 3]` |
| Rental — regular track | Part ד / נספח ב' | rental P&L | נספח ב' (1321) | `[FACT — 1301 PDF + annex checklist]` |
| Foreign rental (15%) | Part ה line 25 | foreign records | §122א; line 25 (15%) | `[FACT — 1301 PDF]` |
| Capital market (שוק ההון) | Part ה (interest/dividends) + part ח (gains) | טופס 867/867א | נספח ג' (1322), ג1/ג2; fields 078/126/142, 256 | `[FACT — 1301 PDF + annex checklist]` |
| Foreign income (חו"ל) | Part ט (field 290/40) **and** in the matching regular fields | foreign statements; foreign-tax proof | נספח ד' | `[FACT — 1301 PDF]` |
| Pensions / allowances (קצבאות) | Part ג line 5 (field 258/272) | טופס 106 from payer/fund | 1301 part ג | `[FACT — 1301 PDF]` |
| ביטוח לאומי taxable receipts | Part ג line 2 (מילואים, דמי לידה, etc.) | ביטוח לאומי certificates | fields 250/270 (self-emp), 194/196 (employee) | `[FACT — 1301 PDF]` |
| Other income | Part ד line 11 / special rates in part ה | varies | 1301 parts ד/ה | `[FACT — 1301 PDF]` |

---

## E. Deductions / credits / adjustments table

| Item | Source / legal hook | Data needed | Common validation | Affects | Tag |
|---|---|---|---|---|---|
| Recognized business expenses | נספח א' (1320) / 6111 | itemized expenses; books | exclude employee-only expenses if also salaried | calculation | `[FACT — kolzchut + 1301]` |
| Credit points (נקודות זיכוי) | §§34, 36, 40, 66 (residency, children, etc.) | residency, children ages, status | base value is year-sensitive | calculation | `[FACT — 1301 part יג; §§ via Tier 3]` |
| Donations (תרומות) | §46 | original receipts to recognized institutions | field 037/237; carry-forward of excess | calculation + documentation | `[FACT — 1301 PDF, §46]` |
| Pension / קופת גמל / קרן השתלמות | §§45, 47 (credit/deduction) | contribution certificates; 134 if excess | employee vs self-employed split | calculation | `[FACT — 1301 parts יב/יד; §§45/47 Tier 3]` |
| ביטוח לאומי paid (non-employment income) | part יב line 54 | NI payment proof | only on non-salary income | calculation | `[FACT — 1301 PDF]` |
| Loss offset (קיזוז הפסדים) | part יא carry-forwards; טופס 1344 | prior-year losses, current losses | track per loss type (business/property/capital/foreign) | calculation | `[FACT — 1301 PDF; 1344 ynet Tier 3]` |
| Depreciation (פחת) | within נספח א'; additional via 1343 | asset register | match to business income | calculation | `[FACT — annex checklist + nevo]` |
| Advances & withholding (מקדמות / ניכוי במקור) | part טו | 106/867/806 certificates | advances auto-included; don't double-enter | calculation | `[FACT — 1301 PDF + refund guide]` |
| Surtax (מס יסף) | §121ב | total taxable income | if > 721,560 ₪ (TY2025) → filing required even if PAYE | readiness + calculation | `[FACT — 1301 PDF; bshcpa Tier 3]` |

---

## F. Submission & status notes

- **Channels.** Default is **online (מקוון)**: complete & transmit on the Authority site (or via a connected representative system). Manual filing is allowed for narrow cases (e.g., both spouses at/above retirement age, or income under thresholds in the online-exemption regulations). `[FACT — kolzchut, Tier 2; תקנות פטור מהגשת דוח עצמאי מקוון]`
- **6111 / short-return transmission.** Transmitted online, then a **barcoded summary page** is printed, signed, and submitted with the return. `[FACT — official guide content]`
- **Signature.** Return signed by "בן הזוג הרשום"; a single signature implies a §144 power of attorney to sign for the spouse. Paid preparers sign under §143 and bear responsibility under §224. `[FACT — 1301 PDF]`
- **Confirmation (אישור הגשה).** Online: ensure a submission-confirmation message is received. Manual: get one of the two copies stamped as the receipt. A return that was *transmitted* (שודר) but not physically submitted is **not** considered filed in the manual track. `[FACT — kolzchut]`
- **Amended return (דוח מתקן).** A reasoned, documented correction request may be filed with the assessing officer up to **4 years from the end of the year in which the return was filed**; procedure governed by **הוראת ביצוע מס הכנסה 3/2018**. (Worked example from the source: corrections for TY2021 close at end of 2026.) `[FACT — kolzchut, Tier 2, citing official הו"ב 3/2018]`
- **Deadlines.**
  - **Statutory base (סעיף 132):** by **30 April** of the following year; **31 May** for filers on full double-entry books / those required to file the online self-employed return. `[FACT — §132(a)/(b) via mirror — verify official text]`
  - **Operational dates, TY2025 (per Authority, via כל זכות):** non-online individuals **29.05.2026**; online individuals **30.06.2026**. Micro-business (עסק זעיר) short return for 2025 extended to **31.05.2026** per an Authority notice. `[FACT — kolzchut, Tier 2]`
  - The Authority issues an annual extension circular; **treat every deadline as year-specific and re-confirm.** `[INFERENCE]`
- **Penalties.** Non-filing without an officer's approval is a **criminal offense** (penalty: up to one year imprisonment, or fine, or both); the officer may also assess income to the best of his judgment (שומה לפי מיטב השפיטה). Interest and linkage (ריבית והצמדה) accrue on tax debt from **1 January** of the following year. `[FACT — kolzchut, Tier 2; alfie Tier 3 corroborates interest start]`

---

## H. Open questions, gaps & conflicts (must-recheck list)

**A. Sources I could not fully fetch (bot-blocked or not opened) — confirmed to exist, content via snippet only:**
1. `gov.il` annual-report **service page** (TY2025) — blocked on direct fetch; relied on its outbound links + snippet. **Re-fetch from an unblocked context.**
2. `gov.il/he/service/itc6111`, `itc-106`, `itc806`, `itc1219` — existence + summary confirmed; not rendered in full.
3. **הוראת ביצוע 3/2018** PDF — linked from כל זכות but not separately opened; the 4-year amendment window and procedure should be read from the instruction itself.
4. Official **פקודה** text for **§131, §132, §135, §121ב, §122, §122א, §45, §46, §47, §240ב** — used Wikisource/commercial mirrors as text witnesses; **verify exact wording and current amendment status on the official legislative database.**

**B. Year-sensitive numbers that need annual re-checking (all currently TY2025 unless noted):**
- 6111 turnover trigger: 256,410 ₪ ex-VAT (1301 form) ≈ 300,000 ₪ incl-VAT (Authority guide).
- מס יסף (surtax) threshold: 721,560 ₪ (§121ב(ה)).
- Foreign-asset reporting threshold: 2,086,000 ₪.
- Securities-sales reporting threshold: 2,810,000 ₪.
- Cross-border transfer reporting: 500,000 ₪ over 12 months.
- Minor-filing threshold: 88,750 ₪ (kolzchut states **TY2023** — likely stale; re-confirm current צו).
- Credit-point base value (e.g., the per-point shekel amount and 2.25/2.75-point bases) — sourced only from a Tier-3 page (TY2024). **`UNVERIFIED — NOT FOUND` against official לוחות עזר.**
- Micro-business "70% of turnover" deemed-income rule and the עסק זעיר 30%-expense option — appear on the 1301 form and kolzchut; verify the governing תקנות for current parameters.

**C. Conflicts / ambiguities surfaced:**
- **Depreciation form number:** Tier-3 "1342" vs official nevo "1343 (ניכוי נוסף בשל פחת)". Higher tier (nevo) preferred; general depreciation otherwise sits in נספח א' (1320). Verify.
- **6111 threshold framing:** ex-VAT vs incl-VAT — reconciled but stated differently across official sources; cite the **form's printed ex-VAT figure** as authoritative for the current year.
- **Statutory vs operational deadline:** §132 says 30 April / 31 May; the Authority's annual extension pushes these later (TY2025: 29.05 / 30.06). Model the **statutory date** and an **Authority-published effective date** as two distinct fields.

**D. Items not enumerated this pass:**
- Full titles of **867א–867ה** sub-certificates (only 867א title confirmed).
- Exact §135 sub-clause for הצהרת הון (only Tier-3 citations of §135(1)/(1)(a)).
- The complete §131(a)(1)–(5x) enumeration of who must file (controlling shareholders §131(a)(5ג), trusts §131(a)(5ב), CFC/CFP, reportable-action/opinion filers, etc.) — partially captured from the 1301 checkboxes; verify the full list against §131 text and the Authority guide.

---

## I. Validation note

This document is **research to support software data-modeling only**. It is **not** tax advice and must **not** drive any real filing, calculation, or client deliverable until reviewed and validated by a **licensed Israeli CPA (רו"ח) or tax advisor (יועץ מס)**. Forms, field codes, thresholds, credit-point values, deadlines, and section numbers change yearly and by amendment; every year-specific value above is tagged and listed in Section H for annual re-verification against the official `gov.il` / `taxes.gov.il` form for the relevant tax year and the current text of פקודת מס הכנסה.
