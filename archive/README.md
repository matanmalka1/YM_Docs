## Scope
This file owns only:
- Archive policy for obsolete or historical documentation.

This file must not contain:
- Active architecture rules, product/domain rules, or TODOs.

Source of truth: reference

# Archive

- `actions-legacy.md` — historical `backend/app/actions/README.md` content before canonical docs migration.
- `actual-domains-v1.md` — historical generated scan of `backend/app/` domain endpoints/services/repositories. Replaced by `docs/project/backend-module-map.md` for current package inventory and `docs/domains/*` for canonical domain behavior; archived because it drifted from code.
- `gap-target-vs-actual-v1.md` — historical target-vs-actual domain gap analysis and build-order plan. Replaced by current domain docs and project/module inventory; archived because it depends on a removed `target-domains-v1.md` and is not active architecture guidance.
- `invoice-legacy.md` — historical `backend/app/invoice/README.md` content before canonical docs migration.
- `migration-phases.md` — historical backend domain migration execution plan. Replaced by
  `docs/project/backend-module-map.md` for current package inventory and `docs/domains/*` for
  canonical domain behavior; archived because the structural migration is complete.

Archived docs are historical only.

Do not use archived docs as active source of truth unless a current mandatory doc explicitly points to them.

When archiving a doc, record what replaced it and why it is no longer canonical.
