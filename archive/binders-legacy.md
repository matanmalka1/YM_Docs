## Scope
This file owns only:
- Historical context for the completed binder lifecycle refactor (T31).
- Preserved rationale and acceptance criteria from the original spec that are no longer needed in canonical docs.

This file must not contain:
- Current-state binder behavior (canonical doc: docs/domains/binders.md).
- New product requirements.

Source of truth: historical

# Binders — Legacy Archive

Archived from: `backend/docs/backend/domains/binder_lifecycle_refactor_spec.md`
Canonical domain doc: `docs/domains/binders.md`

## Summary of completed refactor

The binder lifecycle was refactored from a single mixed `Binder.status` field into two explicit fields:

- `location_status`: `in_office`, `ready_for_handover`, `handed_over`
- `capacity_status`: `open`, `full`

The old `Binder.status` enum, `BinderStatusLog` table, and endpoint names (`ready`, `return`, `pickup`, `return_binder`) were removed as a clean breaking change during active development (no production users at the time).

## Preserved rationale

- `open` is capacity only. Intake eligibility requires `location_status=in_office AND capacity_status=open` — not `open` alone.
- `BinderLifecycleService` is the sole owner of lifecycle transitions. Repository is persistence/query only.
- Available actions are derived from backend and returned in responses; frontend renders, does not decide.
- Hebrew labels live in frontend constants only; backend returns enum values.
- After handover, the same binder is never reused. New material creates a new binder.
- BinderLifecycleLog (one row per changed field) replaced BinderStatusLog (one row per event).

## Old terminology removed

| Old | New |
|-----|-----|
| `ready_for_pickup_at` | `ready_for_handover_at` |
| `returned_at` | `handed_over_at` |
| `pickup_person_name` | `handover_recipient_name` |
| `binder_status_logs` | `binder_lifecycle_logs` |
| `Binder.status` | `location_status` + `capacity_status` |
| endpoints: `ready`, `return`, `pickup`, `return_binder` | current lifecycle endpoints |

## Full original spec

The complete original implementation spec (including prompt, acceptance criteria, and sanity checks) was in `backend/docs/backend/domains/binder_lifecycle_refactor_spec.md`. That file now contains a pointer to this archive. The spec is historical — do not use it as a current implementation prompt without re-auditing against code.
