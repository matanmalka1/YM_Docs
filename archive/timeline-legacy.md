## Scope
This file owns only:
- Historical content from legacy timeline/history docs, preserved for reference.

This file must not contain:
- Current canonical behavior (see docs/domains/timeline.md).

Source of truth: historical

# Timeline — Legacy Archive

Canonical domain doc: `docs/domains/timeline.md`.

This archive consolidates historically useful content from three legacy sources.

---

## From `backend/app/timeline/README.md` (archived 2026-05-29)

> Last audited: 2026-05-08 (operational feed cleanup).

### Explicitly excluded noisy events (as documented)

- `client_info_updated` — belongs in generic audit.
- `reminder_created` — raw reminder setup is not client activity.
- `notification_sent` — automated send rows are noisy. (**Note: code does not match — see F-TL-001 in canonical doc.**)
- `signature_request_created` — lifecycle audit rows are more useful.
- Initial binder `null -> in_office` lifecycle logs when `binder_received` exists.
- Same-value binder lifecycle log rows.

### Test suites (as documented)

- `tests/timeline/api/test_timeline.py`
- `tests/timeline/service/test_timeline_service_get_client_timeline.py`
- `tests/timeline/service/test_timeline_event_builders.py`
- `tests/timeline/service/test_timeline_event_builders_additional.py`
- `tests/timeline/service/test_timeline_tax_builders.py`
- `tests/timeline/service/test_timeline_client_builders.py`
- `tests/timeline/service/test_timeline_operational_policy.py`
- `tests/timeline/service/test_timeline_signature_lifecycle.py`
- `tests/timeline/repository/test_timeline_repository.py`

---

## From `backend/docs/history-vs-timeline.md` (archived 2026-05-29)

### Timeline

Timeline is the operational client activity feed. It is optimized for staff workflow, scanning, filtering, and acting on important client activity.

### AuditTrail

AuditTrail is the accountability/change log: who changed what and when. It is the full source of truth for audited entity changes.

Do not dump all audit records into the timeline. The timeline may later include only selected high-value audit-derived events, but the audit trail remains the complete record.

Generic entity audit UI can later be reused for: `client`, `business`, `charge`, `annual_report`.

---

## From `backend/docs/backend/domains/history-map.md` (archived 2026-05-29)

Last backend endpoint check: 2026-05-29, against `backend/openapi.json` and routers under `backend/app/**/api`.

### Naming convention recommendations (still applicable)

- Use `AuditTrail` for immutable event logs that record who changed what and when.
- Use `StatusHistory` for status transition records only.
- Use `Timeline` for operational or derived chronological product views.
- Use `ReportHistory` only when it means historical report records across years.
- Keep user-facing Hebrew labels flexible (`היסטוריה`, `ציר זמן`), but keep code names precise.

### What should remain separate

- Client business timeline: operational/event timeline with actions and filters.
- Annual report filing timeline: derived lifecycle/deadline visualization.
- Annual report report history table: list of report years, not audit.
- User admin audit logs: security/admin audit.
- Signature request legal audit: legal evidence trail.

### What can share UI later

- Generic entity audit tables for `client`, `business`, `charge`, and `annual_report` can share a table shell, pagination controls, action labels, field-label mapping, and JSON diff formatting.
- Binder lifecycle history and annual report status history can potentially share a status-transition timeline component.

### What should not be merged

- Do not merge timeline with audit trail. Timeline is an operational product view; audit is an accountability record.
- Do not merge signature legal audit into generic audit UI.
- Do not merge user admin audit into entity audit.
