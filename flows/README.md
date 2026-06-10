## Scope

This file owns only:
- Index of cross-domain flow documentation.

This file must not contain:
- Per-domain model or endpoint documentation (belongs in `docs/domains/`).
- Architecture rules (belongs in `docs/architecture/`).

Source of truth: reference

# Cross-Domain Flow Index

Each file documents a complete cross-domain flow as it exists in the code.
Documentation reflects actual behavior, not intended behavior.
Conflicts between code and docs are marked explicitly.

## Critical side-effect flows

| Flow | File | Entry Point |
|------|------|-------------|
| Client + Business creation cascade | [01-client-business-creation-cascade.md](01-client-business-creation-cascade.md) | `CreateClientService.create_from_request()` |
| Annual report status transition | [02-annual-report-status-transition.md](02-annual-report-status-transition.md) | `AnnualReportStatusService.transition_status()` |
| Client freeze / close cascade | [03-client-freeze-close-cascade.md](03-client-freeze-close-cascade.md) | `ClientUpdateService._update_client_record_status()` |

## Cross-domain state flows

| Flow | File | Entry Point |
|------|------|-------------|
| Binder material intake | [04-binder-material-intake.md](04-binder-material-intake.md) | `BinderIntakeService.receive()` |
| VAT work item creation | [05-vat-work-item-creation.md](05-vat-work-item-creation.md) | `create_work_item()` in `vat/services/intake.py` |
| Charge → work queue derived item | [06-charge-work-queue.md](06-charge-work-queue.md) | `charge_items()` in `work_queue/services/billing_items.py` |

## Read-only aggregation flows

| Flow | File | Entry Point |
|------|------|-------------|
| Client status card | [07-client-status-card.md](07-client-status-card.md) | `StatusCardService.get_status_card()` |
| Dashboard overview | [08-dashboard-overview.md](08-dashboard-overview.md) | `DashboardOverviewService.get_overview()` |
| Work queue assembly | [09-work-queue-assembly.md](09-work-queue-assembly.md) | `WorkQueueService.list_items()` |

## Known bugs in flows

- **Flow 3 (Client freeze/close)**: `BinderRepository.close_in_office_by_client_record()` is called but the method does not exist. The entire cascade fails at runtime with `AttributeError`.
