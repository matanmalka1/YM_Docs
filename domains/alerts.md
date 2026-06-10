## Scope

This file owns only:

- Canonical current-state documentation for the alerts domain.

This file must not contain:

- Dashboard endpoint contracts.
- Work-queue domain rules beyond alert input selection.

Source of truth: mandatory

# Alerts

The alerts domain owns internal attention-signal calculation used by the dashboard. After Phase 3, the alert calculation service lives in `backend/app/alerts/services/alert_service.py`; dashboard remains the routed read surface that exposes the computed attention board.

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
- The service fetches one work-queue page of up to 20 items from the attention urgency tiers, filters eligible items, sorts by urgency rank and due date, then caps the result at 7 items.
- `TASK` items require a `due_date` and an attention urgency tier. Non-task source types are eligible after the work-queue urgency filter.
- Client names are attached in bulk through `load_client_profiles`.
- Phase 3 was structural: the class and methods stayed the same while the service moved from dashboard to alerts.

## Error codes

Alerts raises no local `ALERT.*` error codes.

## Known issues

- No standalone alerts API exists. Dashboard is still the only read surface for alert output.

## Decisions (preserved)

1. **Alerts are internal signals.** They are separate from notifications, which send outbound messages, and from dashboard, which is a read model.
2. **Dashboard remains the public surface.** Phase 3 moved ownership of calculation logic only; it did not change API contracts.

## Future / planned

- Add dedicated alert endpoints only if the product needs alert read/acknowledgement outside dashboard.
