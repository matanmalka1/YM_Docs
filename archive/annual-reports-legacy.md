## Scope
This file owns only:
- Historical annual reports domain material archived during canonical domain-doc authoring.

This file must not contain:
- Current implemented annual reports behavior.
- Canonical product, backend, API, or architecture rules.

Source of truth: historical

# Annual Reports Legacy Archive

Canonical current-state doc: `docs/domains/annual-reports.md`.

Archived on 2026-05-29 from:

- `backend/docs/backend/domains/annual_reports/annual_reports_summary.md`
- `backend/docs/backend/domains/annual_reports/report_history_financial_decision.md`
- `backend/docs/backend/domains/annual_reports/source-map.md`

## Preserved historical context

- Annual filing obligations were described as depending on legal/entity profile, income type, and special reporting events.
- Full annual-return forms historically referenced were `1301`, `1214`, and `1215`; `0135` was described as a refund-request form, not a full annual report.
- Partnerships were discussed as requiring partner-level reporting and often `1504`.
- Common annexes historically referenced included schedules A/B/Gimmel/Dalet and forms such as `150`, `1504`, and `6111`.
- The source-map file preserved external government/legal reference URLs for annual-report law, forms, guides, and explanatory material.

## Financial history decision discussion

The legacy decision document discussed whether a client's annual-report history table should remain a status/navigation table or become a multi-year financial comparison table.

Current implementation at archive time:

- `GET /api/v1/clients/{client_record_id}/annual-reports` exists and returns `AnnualReportListResponse`.
- No dedicated endpoint exists for `GET /api/v1/clients/{client_record_id}/annual-reports/history`.
- No endpoint exists for `GET /api/v1/annual-reports/{report_id}/client-history`.

Preserved recommendation, still not implemented:

- If the product needs a stable multi-year financial comparison table, prefer a dedicated financial-history endpoint over inflating generic annual-report list responses with detail-heavy fields.
- Candidate response fields discussed historically: `id`, `tax_year`, `status`, `submitted_at`, `assessment_amount`, `refund_due`, `tax_due`, `total_income`, `total_expenses`, `profit`, and `final_balance`.

## External source map

Historical external references from the legacy source map:

- `https://www.gov.il/he/service/reporting-and-payment-2024-annual-tax-report-for-individuals?trigger=sugg`
- `https://www.gov.il/he/service/reporting-and-payment-2025-annual-tax-report-for-individuals?trigger=sugg`
- `https://www.gov.il/he/service/reporting-and-payment-annual-tax-report-2024-companies?trigger=sugg`
- `https://he.wikisource.org/wiki/%D7%A4%D7%A7%D7%95%D7%93%D7%AA_%D7%9E%D7%A1_%D7%94%D7%9B%D7%A0%D7%A1%D7%94#%D7%A1%D7%A2%D7%99%D7%A3_131`
- `https://www.gov.il/BlobFolder/generalpage/income-tax-guide-knowyourright/he/Guides_IncomeTax_da-2025.pdf`
- `https://www.gov.il/he/service/reporting-and-payment-annual-tax-report-2025-companies?trigger=sugg`
- `https://www.gov.il/he/service/report-and-payment-for-micro-business-owner`
- `https://www.kolzchut.org.il/he/%D7%94%D7%92%D7%A9%D7%AA_%D7%93%D7%95%22%D7%97_%D7%A9%D7%A0%D7%AA%D7%99_%D7%9C%D7%9E%D7%A1_%D7%94%D7%9B%D7%A0%D7%A1%D7%94#.D7.91.D7.A7.D7.A6.D7.A8.D7.94`
- `https://www.gov.il/he/service/itc4435`
- `https://www.gov.il/BlobFolder/generalpage/upload-hidden-files/he/IncomeTax_change-partner-registered.pdf`
- `https://he.wikisource.org/wiki/%D7%AA%D7%A7%D7%A0%D7%95%D7%AA_%D7%9E%D7%A1_%D7%94%D7%9B%D7%A0%D7%A1%D7%94_(%D7%A4%D7%98%D7%95%D7%A8_%D7%9E%D7%94%D7%92%D7%A9%D7%AA_%D7%93%D7%99%D7%9F_%D7%95%D7%97%D7%A9%D7%91%D7%95%D7%9F)`
- `https://www.gov.il/he/pages/previous-annual-tax-report`
- `https://www.bdo.co.il/he-il/blogs-he/%D7%91%D7%9C%D7%95%D7%92-%D7%A2%D7%A1%D7%A7%D7%99%D7%9D-%D7%A7%D7%98%D7%A0%D7%99%D7%9D/%D7%93%D7%95%D7%97-%D7%A9%D7%A0%D7%AA%D7%99-%D7%9E%D7%99,-%D7%9E%D7%AA%D7%99-%D7%95%D7%90%D7%99%D7%9A-%D7%A6%D7%A8%D7%99%D7%9A-%D7%9C%D7%94%D7%92%D7%99%D7%A9`
- `https://www.gov.il/he/service/itc6111`
- `https://www.gov.il/BlobFolder/service/itc6111/he/Service_Pages_Income_tax_annual-report-2022_annual-singular-report-2022_6111-2023-clarification.pdf`
