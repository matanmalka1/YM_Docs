## Scope
This file owns only:
- Historical VAT domain material from legacy `backend/docs/backend/domains/vat_report/` files.

This file must not contain:
- Current-state documentation (see canonical doc).
- Architecture/API rules.

Source of truth: historical

# VAT Reports Legacy

> Archived from:
> - `backend/docs/backend/domains/vat_report/VAT_DEFINITIONS.md`
> - `backend/docs/backend/domains/vat_report/source-map.md`
> - `backend/docs/backend/domains/vat_report/vat_reports_deep_summary.md`
> - `backend/docs/backend/domains/vat_report/vat_reports_domain_summary.md`
>
> Canonical domain doc: `docs/domains/vat-reports.md`
> Archived during canonical doc authoring on 2026-05-29.

## Preserved Historical Definitions

VAT reporting is owned by the legal reporting entity/client, not by a `Business`. A `Business` is an operational activity, branch, or grouping under the legal reporting entity.

Legacy definitions distinguished taxable entities, registered dealers, authorized dealers, exempt dealers, non-profits, financial institutions, and partnerships. These definitions are regulatory context only; the current implementation supports the client/legal-entity shapes and enums present in code.

The legacy VAT report object represented a periodic report for one reporting period. Historical concepts included zero reports, corrected reports, supplemental reports, VAT payment, VAT refund, output VAT, input VAT, capital input VAT, other input VAT, and source documents such as tax invoices, transaction invoices, receipts, credit notes, import entries, and allocation numbers.

## Preserved Historical Rules And Risks

- VAT work should be client-scoped around the legal reporting entity, not business-scoped.
- There must not be more than one VAT report/work item for the same reporting entity and reporting period.
- Exempt dealers do not file regular periodic VAT reports; annual VAT-related obligations were described as a separate concept.
- Legacy docs expected explicit zero-report handling when no activity exists.
- Legacy docs expected correction/amendment support for already filed reports.
- Legacy docs identified regulatory risks if the system does not separate capital input VAT from other input VAT.
- Legacy docs identified allocation-number checks for high-value invoices as important.
- Legacy docs identified timing-basis rules (cash basis vs. accrual/invoice timing) as important for period assignment.
- Legacy docs noted that VAT rate selection depends on the legal tax point.

## Historical Status Lists

The short legacy summary listed desired statuses `draft`, `submitted`, `paid`, `refund_pending`, `refund_approved`, and `zero_report`. These are not current backend enum values.

Current enum values at archive time are documented in `docs/domains/vat-reports.md`.

## Historical Numeric Assumptions

Legacy material included legal/market thresholds and deadlines such as:

- OSEK PATUR ceiling examples.
- Monthly vs. bi-monthly turnover thresholds.
- VAT filing deadlines on the 15th, 19th, or 23rd depending on submission mode.
- Allocation-number threshold examples for 2024, 2025, and 2026.
- VAT rate change to 18% from 2025-01-01.

These numbers were not re-verified as legal advice during archive. Current code reads tax-rule values from configured tax rules where implemented.

## External Source Links From Legacy Source Map

- https://www.kolzchut.org.il/he/%D7%9E%D7%A1_%D7%AA%D7%A9%D7%95%D7%9E%D7%95%D7%AA
- https://www.kolzchut.org.il/he/%D7%A2%D7%95%D7%A1%D7%A7_%D7%A4%D7%98%D7%95%D7%A8
- https://www.kolzchut.org.il/he/%D7%94%D7%92%D7%A9%D7%AA_%D7%93%D7%95%22%D7%97%D7%95%D7%AA_%D7%AA%D7%A7%D7%95%D7%A4%D7%AA%D7%99%D7%99%D7%9D_%D7%95%D7%AA%D7%A9%D7%9C%D7%95%D7%9D_%D7%9E%D7%A1_%D7%A2%D7%A8%D7%9A_%D7%9E%D7%95%D7%A1%D7%A3
- https://www.nevo.co.il/law_html/law00/72813.htm#Seif34
- https://he.wikisource.org/wiki/%D7%A6%D7%95_%D7%9E%D7%A1_%D7%A2%D7%A8%D7%9A_%D7%9E%D7%95%D7%A1%D7%A3_(%D7%A9%D7%99%D7%A2%D7%95%D7%A8_%D7%94%D7%9E%D7%A1_%D7%A2%D7%9C_%D7%A2%D7%A1%D7%A7%D7%94_%D7%95%D7%A2%D7%9C_%D7%99%D7%91%D7%95%D7%90_%D7%98%D7%95%D7%91%D7%99%D7%9F)
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=2
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=3
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=4
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=5
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=6
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=8
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=9
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=10
- https://www.gov.il/he/pages/vat-to-the-new-dealer?chapterIndex=11
- https://www.gov.il/he/pages/faq_chat_israel_invoice
- http://taxes2024.mag.calltext.co.il/magazine/147/pages/68

## Replaced Legacy Files

Each original legacy file now contains only a pointer to the canonical doc and this archive.
