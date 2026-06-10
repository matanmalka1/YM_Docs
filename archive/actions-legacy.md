## Scope

This file owns only:

- Historical actions documentation copied from `backend/app/actions/README.md` before canonical docs migration.

This file must not contain:

- Current source-of-truth rules.

Source of truth: historical

# Actions Module Legacy Notes

Archived source: `backend/app/actions/README.md`.

The actions module defined executable UI action metadata shared across businesses, binders, charges, VAT work items, and annual reports. It also owned cross-domain obligation generation helpers used by client creation/update and business creation.

Historical implementation references before Phase 4:

- Registry: `app/actions/action_registry.py`
- Shared helpers: `app/actions/action_helpers.py`
- Binder actions: `app/actions/binder_actions.py`
- Business actions: `app/actions/business_actions.py`
- Charge actions: `app/actions/charge_actions.py`
- Tax/annual actions: `app/actions/report_deadline_actions.py`
- Obligation workflow: `app/actions/obligation_orchestrator.py`

Those flat module paths became stale in Phase 4, which moved action modules under `app/actions/services/`.

Still-preserved behavior:

- Action descriptors are metadata consumed by other domains.
- Actions has no direct HTTP API and no database tables.
- Target domain endpoints remain responsible for authorization and lifecycle enforcement.
