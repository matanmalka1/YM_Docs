## Scope
This file owns only:
- Open annual-report follow-up tasks from the financial-lines audit.
- Non-canonical implementation notes that still need product, architecture, or engineering decisions.

This file must not contain:
- Canonical annual-report domain rules.
- Architecture rules.
- Completed fixes.

Source of truth: reference

# Annual Reports TODO

Open tasks remaining from the annual-report financial-lines audit after the first clear-fix pass.

Canonical behavior remains in `docs/domains/annual-reports.md`, `docs/domains/audit.md`, and the relevant flow docs. Do not implement product behavior from this file without checking those docs first.

## Already fixed in the first pass

These audit items are intentionally not repeated below:

- #2/#3: financial line repositories now reject unsupported update fields.
- #9/#10: update audit now uses only applied fields; all-`None` updates do not create audit rows.
- #12/#13/#14/#15: income/expense add/update/delete audit payloads now use snapshots; expense snapshots include the important optional fields.
- #16: VAT force-delete counter now increments only on successful scoped delete.
- #22/#24: readiness now uses one report lookup and a named `total_checks`.

## Approved Product Decisions To Implement

- [x] **#18, #20: Treat VAT auto-populate as a financial mutation.**
  - Approved rule: `VatImportService.auto_populate()` must follow the same rules as manual income/expense line mutations.
  - Required behavior: write audit records for created income lines, created expense lines, and deleted/replaced lines when `force=True`.
  - Audit metadata must clearly mark the source as VAT import.
  - Relevant files: `backend/app/annual_reports/services/vat_import_service.py`, `backend/app/annual_reports/services/financial_service.py`, `backend/app/audit/services/entity_audit_writer.py`, `docs/domains/annual-reports.md`.

- [x] **#20, #27: Block all annual-report financial mutations for closed/frozen clients.**
  - Approved rule: closed/frozen clients block manual income/expense create, update, delete, and VAT auto-populate, including `force=True`.
  - Current state: manual create/update/delete and VAT auto-populate are blocked.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`, `backend/app/annual_reports/services/vat_import_service.py`.

- [x] **#37, #38: Handle zero/negative VAT totals explicitly.**
  - Approved rule: zero totals do not create financial lines.
  - Approved rule: negative totals do not create financial lines automatically and must not be silently ignored.
  - Required behavior: return negative totals as `skipped_items` / warnings so credits, refunds, or corrections are visible for review.
  - Relevant file: `backend/app/annual_reports/services/vat_import_service.py`.

- [x] **#41, #42: Return source breakdown for merged VAT categories.**
  - Approved rule: when VAT categories merge into one annual-report category, the response must show which source VAT categories contributed to the generated annual-report category.
  - Preferred audit behavior: include the same breakdown in VAT-import audit metadata.
  - Example shape:
    ```json
    {
      "annual_category": "vehicle",
      "amount": "1200.00",
      "source_vat_categories": {
        "fuel": "800.00",
        "vehicle_maintenance": "400.00"
      }
    }
    ```
  - Relevant file: `backend/app/annual_reports/services/vat_import_service.py`.

- [x] **API contract task: update VAT auto-populate response contract.**
  - Adding `skipped_items` and breakdown fields changes the API response.
  - Required update set: response schema, service tests, API tests, OpenAPI export/baseline, and domain docs.
  - Keep this as a separate implementation task from smaller cleanup fixes.
  - Relevant files: `backend/app/annual_reports/schemas/annual_report_financials.py`, `backend/app/annual_reports/api/annual_report_financials.py`, `backend/app/annual_reports/services/vat_import_service.py`, `backend/openapi.json`, annual-report tests, `docs/domains/annual-reports.md`.

## Engineering Follow-Ups

- [ ] **#1, #5, #6: Rework the financial-line repository contract.**
  - Current issue: income/expense line repositories subclass `BaseRepository` but deliberately block global `get_by_id`, `update`, and `delete`.
  - Suggested direction: a dedicated scoped child-resource repository contract, such as annual-report financial-line repositories with only report-scoped read/update/delete methods.
  - Constraint: avoid a broad abstraction unless it materially reduces duplication and clarifies ownership.
  - Relevant files: `backend/app/annual_reports/repositories/income_repository.py`, `backend/app/annual_reports/repositories/expense_repository.py`, `backend/app/common/repositories/base_repository.py`.

- [ ] **#7, #8: Remove double lookup in financial line update/delete.**
  - Current issue: services load the line for snapshot, then repositories load it again for update/delete.
  - Suggested direction: add repository methods that accept a preloaded scoped entity, such as `apply_updates(line, fields)` and `delete_line(line)`.
  - Relevant files: `backend/app/annual_reports/services/financial_service.py`, income/expense repositories.

- [ ] **#11: Make financial audit scalar normalization stricter.**
  - Current issue: `audit_scalar()` stringifies arbitrary objects.
  - Suggested direction: explicitly handle enum, `Decimal`, `str`, `int`, `float`, `bool`, and `None`; reject or preserve structured values intentionally.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

- [ ] **#17: Consider batch delete for VAT force refresh.**
  - Current issue: `force=True` deletes existing lines one at a time.
  - Constraint: batch deletion must be reconciled with any future audit/side-effect requirements.
  - Relevant file: `backend/app/annual_reports/services/vat_import_service.py`.

- [ ] **#21, #44: Rename or clarify annual-report repository facade vs root repository.**
  - Current issue: both `AnnualReportRepository` and `AnnualReportReportRepository` exist, which is confusing.
  - Suggested direction: document the facade pattern clearly or rename the lower-level report repository during a focused cleanup.
  - Relevant files: `backend/app/annual_reports/repositories/annual_report_repository.py`, `backend/app/annual_reports/repositories/report_repository.py`.

- [ ] **#23, #25: Clean up readiness helper organization.**
  - Current issue: readiness mixes gate logic with user-facing label construction.
  - Suggested direction: keep the documented four gates, but move repeated label/issue construction into a small helper if readiness grows.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

- [ ] **#26: Decide whether missing client record in `_assert_client_allows_create()` should fail closed.**
  - Current issue: missing `ClientRecord` silently passes because only closed/frozen statuses raise.
  - Note: annual reports should not exist without a valid client record, so this is likely a data-integrity guard rather than normal flow behavior.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

- [ ] **#29: Use `self.db` consistently in `invalidate_tax_if_open()`.**
  - Current issue: code constructs `ClientRecordRepository(self.report_repo.db)` instead of `ClientRecordRepository(self.db)`.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

- [ ] **#33: Consider SQL aggregate for recognized expenses.**
  - Current issue: `total_recognized_expenses()` loads all expense lines and sums in Python.
  - Constraint: low priority unless line counts grow.
  - Relevant file: `backend/app/annual_reports/repositories/expense_repository.py`.

- [ ] **#35, #36: Add service/DB-level numeric validation where HTTP schema validation is insufficient.**
  - Current state: HTTP schemas validate amount and recognition-rate ranges, and income has a DB non-negative check. Direct service/repository calls can bypass some validation; expense has no DB amount/rate check.
  - Decision needed: whether to enforce in service only or add DB constraints/migration.
  - Relevant files: `backend/app/annual_reports/schemas/annual_report_financials.py`, income/expense models, `financial_service.py`.

- [ ] **#39, #40: Add VAT-to-annual category mapping coverage and missing-mapping visibility.**
  - Current issue: mapping is hardcoded and unknown VAT categories fall to `OTHER`.
  - Suggested direction: tests for all current VAT categories, plus logging/warning or explicit mapping review for unknown categories.
  - Relevant file: `backend/app/annual_reports/services/vat_import_service.py`.

- [ ] **#43: Split `AnnualReportFinancialService` by responsibility.**
  - Current issue: one service owns CRUD, audit, summary, tax calculation, readiness, tax save, and invalidation.
  - Suggested direction: split only when actively touching enough of the module to reduce complexity, e.g. financial-line, summary, tax, and readiness services.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

- [ ] **#50: Add precise typing to financial snapshot helpers.**
  - Current issue: snapshot helpers return untyped `dict`.
  - Relevant file: `backend/app/annual_reports/services/financial_service.py`.

## Wider API / Money Follow-Ups

- [ ] **#31, #32: Remove float conversions from money paths.**
  - Current issue: financial summary and tax calculation convert some `Decimal` values to `float` before response model validation.
  - Constraint: this may affect API output details and tax engine assumptions; handle as a focused API/finance precision task.
  - Relevant files: `backend/app/annual_reports/services/financial_service.py`, `backend/app/annual_reports/schemas/annual_report_financials.py`, tax/NI engines.

## Low-Priority Observations

- [ ] **#30: Re-evaluate the `get_financial_summary()` wrapper only if refactoring.**
  - Current state: thin public wrapper around `_build_financial_summary()`.

- [ ] **#49: Remove `__all__` only if the module is otherwise being cleaned up.**
  - Current state: harmless metadata.
