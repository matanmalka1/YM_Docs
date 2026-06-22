## Scope

This file owns only:
- The project-level rules for the standalone tax-rules configuration package.
- Where to add or read official Israeli tax rules used by the backend.

This file must not contain:
- Per-domain workflow behavior.
- Backend API, UI, or persistence architecture rules.

Source of truth: mandatory

# Tax Rules Config

`backend/tax_rules_config/` is the source of truth for official tax-rule constants, calendars,
obligation rules, and VAT deduction policy used by the application.

## Entry Points

- Application code consumes the package through `backend/app/tax_calendar/integrations/tax_rules_registry.py`
  or directly through `tax_rules.registry` when a domain has an explicit integration.
- Package docs live in `backend/tax_rules_config/README.md`.
- Package tests live in `backend/tax_rules_config/tests/`.

## Rules

- Do not add hardcoded tax deadlines, VAT rates, thresholds, or obligation constants in `backend/app/`
  when the value belongs in `tax_rules_config`.
- Add a new tax year by adding the year-specific calendar/financial constants and registering them in
  `tax_rules.registry`.
- Put exceptional deadline deferrals in the package's exceptions module, not inline in app services.
- Every obligation rule must carry `source_ids` that point to entries in `tax_rules.sources`.
- Domain services may snapshot or materialize tax-rule output, but the official rule definition stays
  in `tax_rules_config`.

## Verification

Run package tests from the backend repo with the repo virtualenv:

```bash
cd backend/tax_rules_config
../.venv/bin/python -m pytest tests/ -q
```

Do not use global `python` or `python3`.

## Known Integration Gaps

- Some application code still has hardcoded tax-calendar or obligation assumptions. When touching
  tax deadlines, VAT rates, thresholds, or obligation resolution, check for an existing registry
  value first and report any drift you cannot safely fix in the same task.
