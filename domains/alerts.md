## Scope

This file owns only:

- Canonical current-state documentation for the alerts domain.

This file must not contain:

- Dashboard endpoint contracts.
- Work-queue domain rules beyond alert input selection.

Source of truth: mandatory

# Alerts

The alerts domain owns internal attention-signal calculation used by the dashboard. It is a logical domain only: there is no `backend/app/alerts/` module. The calculation lives in `backend/app/dashboard/services/dashboard_attention_service.py` (`DashboardAttentionService`), and dashboard is the routed read surface that exposes the computed attention board.

Last verified against code + current OpenAPI export: 2026-06-10.

## Endpoints

No dedicated alerts endpoints exist. Alert data is currently exposed through `GET /api/v1/dashboard/overview` as the dashboard `attention` payload.

## Model & fields

Alerts has no database tables and no local Pydantic schemas. It returns dictionaries shaped for `AttentionBoardItem` in the dashboard response schema.

## Enums / statuses

Alerts references work-queue enums:

| Enum | Values used |
|------|-------------|
| `WorkQueueUrgency` | `OVERDUE`, `APPROACHING`, `IMPORTANT`, `UPCOMING` |
| `WorkQueueSourceType` | `VAT_WORK_ITEM`, `ANNUAL_REPORT`, `ADVANCE_PAYMENT`, `CHARGE`, `BINDER`, `TASK` |

Attention eligibility uses `{OVERDUE, APPROACHING, IMPORTANT}`.

## Domain rules & invariants

- Alerts are advisor-only. `DashboardAttentionService.build(...)` returns an empty list unless `user_role == UserRole.ADVISOR`.
- Alerts are read-only and derived from `WorkQueueService`; no alert rows are persisted.
- `DashboardAttentionService.build(...)` fetches one work-queue page of up to 20 items (`_ATTENTION_FETCH_PAGE_SIZE`) from the attention urgency tiers, filters eligible items, sorts by urgency rank and due date, then caps the result at 7 items (`_MAX_ITEMS`). Cite: `backend/app/dashboard/services/dashboard_attention_service.py:120-144`.
- `TASK` items require a `due_date` and an attention urgency tier. Non-task source types are eligible after the work-queue urgency filter.
- Client names are attached in bulk through `load_client_profiles` (`_attach_client_names`).

## Error codes

Alerts raises no local `ALERT.*` error codes.

## Known issues

- No standalone alerts API exists. Dashboard is still the only read surface for alert output.

## Decisions (preserved)

1. **Alerts are internal signals.** They are separate from notifications, which send outbound messages, and from dashboard, which is a read model.
2. **Dashboard remains the public surface and the home of the calculation.** Alert logic lives in `DashboardAttentionService` inside the dashboard module; there is no separate `app/alerts/` module and no dedicated alert API contract.

## Future / planned

- Add dedicated alert endpoints only if the product needs alert read/acknowledgement outside dashboard.
