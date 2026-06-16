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

Action descriptors are **availability metadata, not executable commands**. They
declare which action a resource currently exposes and how to present it; they do
**not** carry a target endpoint/method/payload. The frontend reads `key` to know
an action is available and routes it to a typed API client/hook. Unknown keys
are not executed. Target endpoints/services remain the source of truth for
authorization and business validation.

Primary output is `ActionDescriptor` (`backend/app/core/action_schemas.py`),
built via `link_action` / `mutation_action` / `modal_action`
(`backend/app/core/action_builders.py`):

| Field | Type | Notes |
|-------|------|-------|
| `key` | str | stable action key — the only field execution depends on |
| `label` | str | user-facing label |
| `type` | str | `link`, `mutation`, or `modal` (UI dispatch hint) |
| `route` | str \| None | client route for `link` actions |
| `task_id` | int \| None | task id for work-queue task actions |
| `confirm` | bool | whether a confirm dialog is shown |
| `confirm_title` | str \| None | confirm dialog title |
| `confirm_message` | str \| None | confirm dialog body |
| `variant` | str | `primary` / `secondary` / `danger` |
| `disabled` | bool | action shown but not actionable |
| `disabled_reason` | str \| None | why the action is disabled |

No `endpoint`, `method`, `payload`, or `payload_schema` field is emitted: the
frontend never executes a descriptor-supplied request.

## Modules

Action service modules:

- `action_registry.py` — aggregate import surface for domain action factories.
- `binder_actions.py` — binder action factories.
- `business_actions.py` — business action factories.
- `charge_actions.py` — charge action factories.
- `vat_report_actions.py` — VAT work-item action factories.
- `report_deadline_actions.py` — annual-report action factories.
- `obligation_orchestrator.py` — cross-domain obligation generation used by client creation/update and business creation flows.

Shared builders/schema live in `backend/app/core/`: `action_schemas.py`
(`ActionDescriptor`) and `action_builders.py`
(`link_action`/`mutation_action`/`modal_action`).

## Enums / statuses

Actions defines no local enum classes. Action factories branch on statuses owned by their target domains, such as binder state, business status, charge status, VAT work-item status, and annual-report status.

## Domain rules & invariants

- Callers should import the aggregate factories from `app.actions.services.action_registry` when possible.
- Action descriptors are availability metadata, not executable commands. They carry no endpoint/method/payload; the frontend dispatches known `key`s to typed API clients/hooks and the target endpoint still owns authorization and business validation.
- Domain action factories must not become the source of truth for lifecycle rules. They expose which actions are available; target services enforce transitions.
- `generate_client_obligations(...)` preserves the legacy integer return for existing callers.
- `generate_client_obligations_result(..., best_effort=True)` returns explicit partial-success detail with `deadlines_created`, `reports_created`, and `errors`.
- Phase 4 was structural: flat modules moved under `app/actions/services/` and importers were repointed without changing action definitions.

## Error codes

Actions has no HTTP surface and no local `ACTION.*` error-code namespace. Errors surface through the endpoint or service that executes the action target.

## Frontend execution contract

Action descriptors are availability metadata only. The frontend treats each
descriptor as a capability flag (by `key`) and executes the action through its
own per-domain typed API client/hook. Descriptors carry no
`endpoint`/`method`/`payload` — there is no generic execution path. Unknown
action keys are ignored, never executed.

## Decisions (preserved)

1. **Actions is an application layer, not a business domain with persistence.** It centralizes command metadata and cross-domain orchestration helpers.
2. **No compatibility imports.** After Phase 4, callers use `app.actions.services.*`; old flat `app.actions.*` imports are not preserved.

## Future / planned

- Add `actions/api/`, `actions/models/`, or `actions/schemas/` only if a real action registry/execution endpoint is introduced.
