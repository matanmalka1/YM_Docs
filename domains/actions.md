## Scope

This file owns only:

- Canonical current-state documentation for the actions domain.

This file must not contain:

- Frontend execution rules.
- Business behavior owned by the target domains of each action.

Source of truth: mandatory

# Actions

The actions domain owns backend action metadata factories and cross-domain action orchestration helpers used by UI-facing responses. After Phase 4, action modules live under `backend/app/actions/services/`; there is no actions router, model, repository, or database table.

Last verified against code + current OpenAPI export: 2026-06-10.

## Endpoints

No actions endpoints exist. Actions are attached by other domains to their response payloads as `available_actions` or similar fields.

## Model & fields

Actions has no database model.

Primary output is an action descriptor dictionary built by `build_action(...)` in `backend/app/actions/services/action_helpers.py`:

| Field | Type | Notes |
|-------|------|-------|
| `id` | str | frontend action id |
| `key` | str | stable action key |
| `label` | str | user-facing label |
| `method` | str | one of `get`, `post`, `put`, `patch`, `delete` |
| `endpoint` | str | relative API endpoint |
| `payload` | dict | optional |
| `confirm` | dict | optional confirm-dialog metadata |

`build_confirm(...)` returns confirm metadata with `title`, `message`, `confirm_label`, `cancel_label`, and optional text inputs.

## Modules

Action service modules:

- `action_helpers.py` — shared action/confirm builders.
- `action_registry.py` — aggregate import surface for domain action factories.
- `binder_actions.py` — binder action factories.
- `business_actions.py` — business action factories.
- `charge_actions.py` — charge action factories.
- `vat_report_actions.py` — VAT work-item action factories.
- `report_deadline_actions.py` — annual-report action factories.
- `obligation_orchestrator.py` — cross-domain obligation generation used by client creation/update and business creation flows.

## Enums / statuses

Actions defines no local enum classes. Action factories branch on statuses owned by their target domains, such as binder state, business status, charge status, VAT work-item status, and annual-report status.

## Domain rules & invariants

- Callers should import the aggregate factories from `app.actions.services.action_registry` when possible.
- `build_action(...)` validates the HTTP method and raises `ValueError` for unsupported methods.
- Action descriptors are transport-oriented metadata; the target endpoint still owns authorization and business validation.
- Domain action factories must not become the source of truth for lifecycle rules. They expose available commands; target services enforce transitions.
- `generate_client_obligations(...)` preserves the legacy integer return for existing callers.
- `generate_client_obligations_result(..., best_effort=True)` returns explicit partial-success detail with `deadlines_created`, `reports_created`, and `errors`.
- Phase 4 was structural: flat modules moved under `app/actions/services/` and importers were repointed without changing action definitions.

## Error codes

Actions has no HTTP surface and no local `ACTION.*` error-code namespace. Errors surface through the endpoint or service that executes the action target.

## Known issues

- No generic action execution API exists. The frontend executes action target endpoints directly.

## Decisions (preserved)

1. **Actions is an application layer, not a business domain with persistence.** It centralizes command metadata and cross-domain orchestration helpers.
2. **No compatibility imports.** After Phase 4, callers use `app.actions.services.*`; old flat `app.actions.*` imports are not preserved.

## Future / planned

- Add `actions/api/`, `actions/models/`, or `actions/schemas/` only if a real action registry/execution endpoint is introduced.
