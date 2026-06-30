## Scope
This file owns only:
- Canonical current-state documentation for the timeline domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Timeline

The timeline domain provides a unified, per-client operational activity feed. It aggregates meaningful business milestones from multiple source domains (binders, charges, invoices, annual reports, documents, signature requests, notifications) into a single reverse-chronological event stream, normalized to a shared `TimelineEvent` schema. The domain owns no persistent database models of its own; all data is composed at query time.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/clients/{client_record_id}/timeline | Fetch paginated, unified client timeline |

Path exists in `backend/openapi.json` (operationId: `get_client_timeline_api_v1_clients__client_record_id__timeline_get`).

### Auth / roles

Required roles: `ADVISOR`, `SECRETARY` (enforced via `require_role` dependency at router level — `backend/app/timeline/api/timeline.py:8-12`).

### Query parameters

| Param | Type | Default | Constraints | Notes |
|-------|------|---------|-------------|-------|
| `page` | int | 1 | ≥ 1 | |
| `page_size` | int | 20 | 1–200 | Max enforced in OpenAPI schema |

### Response: `ClientTimelineResponse`

| Field | Type | Required |
|-------|------|----------|
| `client_record_id` | int | yes |
| `events` | list[TimelineEvent] | yes |
| `page` | int | yes |
| `page_size` | int | yes |
| `total` | int | yes — count of all events before pagination |

## Model & fields

The domain defines **no database models**. It owns only response/transfer schemas.

### `TimelineEvent` (`backend/app/timeline/schemas/timeline.py:8`)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `event_type` | str | yes | String constant; see Event types below |
| `timestamp` | ApiDateTime | yes | ISO-8601 datetime |
| `binder_id` | int \| null | no | Set for binder events |
| `charge_id` | int \| null | no | Set for charge/invoice events |
| `description` | str | yes | Hebrew human-readable label |
| `metadata` | dict[str, Any] | no (default `{}`) | Event-specific key/value payload |

`binder_id` and `metadata` have no default in Pydantic schema — they are optional in the OpenAPI contract.

## Enums / statuses

The domain has no enum models of its own. Event types are string constants assembled by builder modules. The full set currently produced:

### Binder events (`backend/app/timeline/services/timeline_binder_event_builders.py`)

| `event_type` | Trigger |
|---|---|
| `binder_received` | Binder has `received_at` or `period_start` (live builder) |
| `binder_handed_over` | Binder `handed_over_at` is set (live builder) |
| `binder_lifecycle_change` | `EntityAuditLog` `binder.*` row for a capacity/location transition (`marked_full`, `reopened`, `marked_ready_for_handover`, `reverted_ready`); since Phase 5 sourced from `EntityAuditLog`, not the legacy `BinderLifecycleLog`. `binder.created`/`material_received`/`handed_over` are excluded — reception and handover are covered by the live builders above. |

### Charge / invoice events (`backend/app/timeline/services/timeline_charge_event_builders.py`)

| `event_type` | Trigger |
|---|---|
| `charge_created` | Always, per charge |
| `charge_issued` | `charge.issued_at` is set |
| `charge_paid` | `charge.paid_at` is set |
| `invoice_attached` | Invoice found in `invoice_map` for the charge |

### Annual report events (`backend/app/timeline/services/timeline_tax_builders.py`)

| `event_type` | Trigger |
|---|---|
| `annual_report_status_changed` | One row per `EntityAuditLog` entry with action `annual_report.status_changed` |

### Client / document events (`backend/app/timeline/services/timeline_client_builders.py`)

| `event_type` | Trigger |
|---|---|
| `client_created` | Per client `LegalEntity` resolved from the client record |
| `document_uploaded` | Per non-deleted `PermanentDocument` across all client businesses |
| `signature_request_sent` | `SignatureAuditEvent.event_type == "sent"` |
| `signature_request_signed` | `SignatureAuditEvent.event_type == "signed"` |
| `signature_request_declined` | `SignatureAuditEvent.event_type == "declined"` |
| `signature_request_canceled` | `SignatureAuditEvent.event_type == "canceled"` |
| `signature_request_expired` | `SignatureAuditEvent.event_type == "expired"` |

### Notification events (`backend/app/timeline/services/timeline_notification_event_builders.py`)

| `event_type` | Trigger |
|---|---|
| `notification_sent` | Notification with status `SENT` |
| `notification_failed` | Notification with status `FAILED` |

### Hebrew label maps (`backend/app/timeline/labels.py`)

| Map constant | Purpose |
|---|---|
| `BINDER_LIFECYCLE_HE` | Maps binder lifecycle values to Hebrew |
| `CHARGE_TYPE_HE` | Maps `ChargeType` enum values to Hebrew |
| `ANNUAL_REPORT_STATUS_HE` | Maps annual report status values to Hebrew |
| `DOCUMENT_TYPE_HE` | Maps permanent document types to Hebrew |
| `SIGNATURE_REQUEST_TYPE_HE` | Maps signature request types to Hebrew |
| `NOTIFICATION_TRIGGER_HE` | Maps notification trigger types to Hebrew |
| `REMINDER_TYPE_HE` | Maps reminder types to Hebrew (defined; not used in timeline builders) |

## Domain rules & invariants

All rules sourced from `backend/app/timeline/services/timeline_service.py` unless noted.

1. **Client existence check first.** `get_client_timeline` resolves the client record by `client_record_id` before any aggregation. Missing record raises `TIMELINE.CLIENT_NOT_FOUND`. (`timeline_service.py:60-62`)

2. **Businesses scoped by legal entity.** Business IDs are collected by joining on `client_record.legal_entity_id`, filtering `deleted_at IS NULL`. Event sources requiring business scope (charges, documents) use these IDs. (`timeline_service.py:63-69`)

3. **Full aggregation, then sort, then paginate.** All events from all sources are collected in memory, sorted descending by `timestamp`, then sliced by `(page-1)*page_size : page*page_size`. `total` reflects the pre-slice count. (`timeline_service.py:106-109`)

4. **Per-client bulk safety cap: `_TIMELINE_BULK_LIMIT = 500`.** Applied to high-volume sources: charges, permanent documents, signature lifecycle events, annual report status audit events, and notifications. Events beyond this cap per source are silently truncated — oldest events disappear without notice. (`timeline_service.py:39`, `timeline_repository.py:11`)

5. **Binder lifecycle source (Phase 5).** Binder lifecycle events are read from `EntityAuditLog` `binder.*` rows via an explicit action allowlist (`marked_full`, `reopened`, `marked_ready_for_handover`, `reverted_ready`) — `binder.created`, `binder.material_received`, and `binder.handed_over` are excluded so reception/handover stay single-sourced from the live `binder_received`/`binder_handed_over` builders. This replaced the legacy `BinderLifecycleLog` reader and its old-value/new-value dedup heuristics. (`timeline_service.py`)

6. **Annual report events from audit, not report row.** Events are emitted per `EntityAuditLog` row where `entity_type = annual_report` and `action = annual_report.status_changed` (one event per audited status transition), not from the current report state. (`timeline_repository.py:63-110`, `timeline_service.py:206-207`)

7. **Signature events from audit log, not request row.** Events are emitted per `SignatureAuditEvent` row for the lifecycle types `sent`, `signed`, `declined`, `canceled`, `expired`. (`timeline_repository.py:41-59`)

8. **Notifications: SENT and FAILED only.** Notifications with other statuses (e.g. `PENDING`) are excluded. (`timeline_service.py:130-141`) Both `notification_sent` and `notification_failed` events are included in the timeline.

9. **Read-only, informational only.** Timeline events carry no executable action fields. The endpoint is GET-only.

10. **No `notification_id`-scoped client check on notification fetch.** Notifications are filtered only by `client_record_id` at the repository level.

## Error codes

| Code | HTTP | Condition |
|------|------|-----------|
| `TIMELINE.CLIENT_NOT_FOUND` | 404 | `client_record_id` does not match any active client record |

Registry: `docs/backend/error-codes.md`.

## Known issues

No open known issues.

## Resolved issues

- **F-039 / F-TL-001** (2026-06-05): Legacy `backend/app/timeline/README.md` excluded `notification_sent` as noisy. Code intentionally emits both `notification_sent` and `notification_failed`. Decision: keep both — failure signals are useful. Domain doc updated to reflect current behaviour. Legacy README is a pointer only.
- **F-040 / F-TL-002** (2026-06-05): `decline_reason` for `signature_request_declined` events was read from the live `SignatureRequest.decline_reason` column (mutable, not a snapshot). Fixed: `signer_actions.py` already stores `reason` in `SignatureAuditEvent.notes` at decline time. `timeline_client_builders.py` now reads `audit_event.notes` instead.

## Decisions (preserved)

From `backend/docs/history-vs-timeline.md` (still accurate):

- **Timeline ≠ audit trail.** Timeline is an operational client activity feed, optimized for staff scanning, filtering, and acting. Audit trail is the accountability/change log (who changed what and when). Do not merge them.
- **Selective event inclusion.** Not all audit rows should appear in the timeline. Only meaningful business milestones are included. Generic entity-change rows (e.g. `client_info_updated`) belong in the generic audit UI, not in the timeline.

From `backend/app/timeline/README.md` (still accurate):

- **No persistent models.** The timeline domain is intentionally read-only and stateless — it composes data from other domains at query time.
- **Cross-domain event builders are split by source domain** for maintainability (`binder_event_builders`, `charge_event_builders`, `tax_builders`, `client_builders`, `notification_event_builders`).

## Future / planned

The following items appear in legacy notes or README but are **not yet implemented**:

- **Tax calendar upcoming deadlines** in the timeline — not currently a timeline event source. (`backend/app/timeline/README.md:117`)
- **VAT report lifecycle events** — not currently a timeline event source. (`backend/app/timeline/README.md:118`)
- **Advance payment lifecycle events** — not currently a timeline event source. (`backend/app/timeline/README.md:119`)
- **Correspondence entries** in the timeline — not currently a timeline event source. (`backend/app/timeline/README.md:120`)
- **Generic audit UI reuse** for `client`, `business`, `charge`, `annual_report` — audit tables can later share a table shell and pagination controls across these entities, but timeline must remain separate. (`backend/docs/history-vs-timeline.md:24-27`)
- **Silent truncation warning** — per `_TIMELINE_BULK_LIMIT`, clients with more than 500 events per source will have older events silently dropped. A future improvement would surface a warning in the API response rather than truncating silently.

## Historical notes

Legacy files for this domain:
- `backend/app/timeline/README.md` — replaced by pointer; historical content preserved at `docs/archive/timeline-legacy.md`.
- `backend/docs/history-vs-timeline.md` — replaced by pointer; still reference-grade.
- Historical frontend history/audit surface map preserved at `docs/archive/timeline-legacy.md`.
