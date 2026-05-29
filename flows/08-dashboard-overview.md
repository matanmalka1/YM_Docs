## Scope

This file owns only:
- Documentation of the dashboard overview aggregation as it exists in the code.

Source of truth: reference

# Flow: Dashboard Overview

## 1. Trigger

`GET /dashboard/overview`

## 2. Entry Point in Code

```
backend/app/dashboard/services/dashboard_overview_service.py
  DashboardOverviewService.get_overview(reference_date, user_role)
```

Router: `backend/app/dashboard/api/dashboard_overview.py`

## 3. Sequence

1. Resolve `reference_date` (default: `israel_today()`).
2. `TaxStatusStatsService.build(reference_date)` — VAT stats; runs for all roles.
3. **ADVISOR-only** branches (SECRETARY gets empty/0 for all of these):
   a. `DashboardAttentionService.build(user_role, reference_date)` — attention items.
   b. `_open_charges_stats()` → `ChargeRepository.open_charges_stats()` — count + total amount of ISSUED charges.
   c. `_build_quick_actions(reference_date)` — returns `[]` (method body is empty, placeholder only).
   d. `AdvisorTodayService.build(reference_date)` — deadline items due today or soon.
   e. `RecentActivityService.build()` — recent audit log activity.
4. `ClientRecordRepository.count()` — checks if any clients exist (for `is_empty` flag).
5. Return assembled dict.

**Note**: `NotificationRepository` is instantiated in `__init__` but not called in `get_overview()`.

## 4. Domains Involved

| Domain | Role |
|--------|------|
| `dashboard` | Owns the service and aggregation |
| `vat_reports` | VAT stats (via TaxStatusStatsService) |
| `charge` | Open charges count + amount |
| `annual_reports` | Attention items, advisor today deadlines |
| `binders` | Attention items |
| `clients` | is_empty check, active client scope |
| `audit` | Recent activity |
| `notification` | Instantiated but unused in this path |

## 5. Side Effects

None. Read-only.

## 6. Transaction Boundaries

Read-only. All in one DB session. No writes.

## 7. Idempotency

N/A — read-only.

## 8. Locks / Concurrency

None.

## 9. Preconditions

None. No client or entity required.

## 10. Blockers / Validation Failures

None at read time.

## 11. Derived State

All data computed from live DB state. Nothing cached.

| Field | Source |
|-------|--------|
| `is_empty` | `ClientRecordRepository.count() == 0` |
| `open_charges_count` | Count of ISSUED charges |
| `open_charges_amount_ils` | Sum of ISSUED charge amounts, formatted as ₪ string |
| `vat_stats` | Aggregated from VatWorkItem table |
| `attention` | Cross-domain items from DashboardAttentionService |
| `advisor_today` | Deadline items from AdvisorTodayService |
| `recent_activity` | Audit log entries |
| `quick_actions` | Always `[]` (not implemented) |

## 12. Role-Based Access

| Field | ADVISOR | SECRETARY |
|-------|---------|-----------|
| `vat_stats` | ✓ | ✓ |
| `is_empty` | ✓ | ✓ |
| `open_charges_*` | ✓ | 0 / null |
| `attention` | ✓ | [] |
| `quick_actions` | [] | [] |
| `advisor_today` | ✓ | `{"deadline_items": []}` |
| `recent_activity` | ✓ | [] |

## 13. N+1 Risk

Not verified for `DashboardAttentionService` and `AdvisorTodayService` internals — those services were not fully read for this documentation. Review them separately if performance issues arise.

## 14. Tests

- `tests/dashboard/api/test_dashboard_extended.py`
- `tests/dashboard/api/test_dashboard_tax.py`
- `tests/dashboard/repository/test_dashboard_overview_repository.py`

**GAP**: `_build_quick_actions` returns `[]` unconditionally — dead code placeholder. No test documents this.

## 15. Documentation Target

- `docs/domains/dashboard.md` — overview endpoint, role access, data sources
