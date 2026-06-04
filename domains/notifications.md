## Scope
This file owns only:
- Canonical current-state documentation for the notification domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Notification

The notification domain records outbound client-communication attempts, renders preview/send content for supported triggers, applies domain policy before delivery, and stores the resulting audit trail. In the current codebase, manual HTTP flows and the single automatic binder-handover flow both persist rows in `notifications`; real delivery is email-only.

Last verified against code + backend/openapi.json: 2026-06-04.

## Endpoints

Router prefix is `/notifications` under `/api/v1` (`backend/app/notification/api/notifications.py:33-37`). All four paths below exist in `backend/openapi.json`.

Auth:
- Notification endpoints intentionally allow both `ADVISOR` and `SECRETARY` at the router level (`backend/app/notification/api/notifications.py:33-36`).

| Method | Path | Purpose |
|--------|------|---------|
| GET | /api/v1/notifications | List persisted notification rows with filters and pagination |
| GET | /api/v1/notifications/summary | Return counts by persisted status (`pending`, `sent`, `failed`, `skipped`) plus `total` |
| POST | /api/v1/notifications/preview | Render a manual notification preview or return `blocked` with a reason |
| POST | /api/v1/notifications/send | Execute a manual send attempt with idempotency protection |

## Model & fields

Model: `Notification` in `backend/app/notification/models/notification.py:88-156`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | autoincrement |
| `client_record_id` | FK -> `client_records.id` | no | primary anchor for all notifications |
| `business_id` | FK -> `businesses.id` | yes | optional business context |
| `binder_id` | FK -> `binders.id` | yes | optional binder context |
| `annual_report_id` | FK -> `annual_reports.id` | yes | optional annual-report context |
| `signature_request_id` | FK -> `signature_requests.id` | yes | optional signature context |
| `entity_type` | `String` | yes | generic anchor type such as `charge`, `annual_report`, `signature_request`, `vat_work_item` |
| `entity_id` | int | yes | generic anchor id |
| `trigger` | `NotificationTrigger` | no | required business trigger |
| `channel` | `NotificationChannel` | no | stored channel enum; current send paths use email only |
| `recipient` | `String` | yes | `null` when a send is skipped for missing contact |
| `content_snapshot` | `Text` | no | rendered body saved as immutable audit snapshot |
| `subject_snapshot` | `Text` | yes | rendered subject snapshot |
| `status` | `NotificationStatus` | no | defaults to `pending` |
| `sent_at` | datetime | yes | stamped on successful delivery |
| `failed_at` | datetime | yes | stamped on failed delivery |
| `error_message` | `Text` | yes | delivery failure reason |
| `retry_count` | `SmallInteger` | no | defaults to `0`; current services do not increment it |
| `idempotency_key` | `String` | yes | request dedupe key |
| `request_hash` | `String` | yes | payload hash paired with the idempotency key |
| `triggered_by` | FK -> `users.id` | yes | `null` means system-triggered |
| `created_at` | datetime | no | defaults to `utcnow` |

Indexes:
- `idx_notification_status`, `idx_notification_client_record_status`, `idx_notification_business_status`, `idx_notification_created_at`, `idx_notification_trigger`, `idx_notification_triggered_by`, `idx_notification_idempotency`, `idx_notification_signature_request` (`backend/app/notification/models/notification.py:147-156`).

Schema notes:
- `NotificationSendRequest.channel` accepts only `email` or `null`; omitted channel defaults to email in the service (`backend/app/notification/schemas/notification_schemas.py:32-42`, `backend/app/notification/services/notification_send_service.py:290`).
- Read responses add derived `client_name`, `business_name`, `trigger_label`, and `domain_label`; those are enrichment fields, not model columns (`backend/app/notification/schemas/notification_schemas.py:65-107`, `backend/app/notification/services/notification_service.py:29-43,76-137`).

## Enums / statuses

### NotificationChannel

Defined in `backend/app/notification/models/notification.py:27-30`.

| Value | Current meaning |
|-------|-----------------|
| `email` | Only channel used by the manual and automatic send services (`backend/app/notification/services/notification_send_service.py:290`, `backend/app/notification/services/notification_auto_send_service.py:163-217`) |
| `whatsapp` | Persistable enum value, but no notification-domain send path currently uses it (`backend/app/notification/models/notification.py:27-30`) |

### NotificationStatus

Defined in `backend/app/notification/models/notification.py:32-36`.

| Value | Meaning |
|-------|---------|
| `pending` | Record created before delivery or after a prior interrupted attempt |
| `sent` | Delivery succeeded and `sent_at` was stamped |
| `failed` | Delivery failed and `failed_at` / `error_message` were stamped |
| `skipped` | Policy allowed the attempt but no recipient email was resolved |

### NotificationTrigger

Defined in `backend/app/notification/models/notification.py:39-52`.

| Value |
|-------|
| `binder_ready_for_handover` |
| `binder_missing_documents` |
| `binder_general_reminder` |
| `invoice_issued` |
| `payment_reminder` |
| `vat_documents_reminder` |
| `annual_report_documents_request` |
| `annual_report_client_reminder` |
| `signature_request_sent` |
| `signature_request_reminder` |
| `client_missing_information` |
| `client_documents_request` |
| `client_general_message` |

Trigger labels and domain labels used in list responses are defined in `TRIGGER_LABELS` / `TRIGGER_DOMAIN` (`backend/app/notification/models/notification.py:55-85`).

## Domain rules & invariants

- `GET /api/v1/notifications` accepts the filter set `client_record_id`, `business_id`, `status`, `trigger`, `channel`, `triggered_by`, `date_from`, `date_to`, then sorts by `created_at DESC` and paginates with total count from the repository (`backend/app/notification/api/notifications.py:40-72`, `backend/app/notification/repositories/notification_repository.py:103-150`).
- `page_size` is intentionally restricted to `25` or `50`; the query contract publishes those values as an enum (`backend/app/notification/api/notifications.py:28-30,51-52`).
- Manual preview/send reject the auto-only trigger `binder_ready_for_handover`. Annual-report triggers require `entity_id`; `invoice_issued`, `payment_reminder`, `vat_documents_reminder`, `signature_request_sent`, and `signature_request_reminder` also require `entity_id` (`backend/app/notification/services/notification_send_service.py:20-43,114-124,192-203`).
- Preview/send first load the `ClientRecord`; missing clients raise `CLIENT.NOT_FOUND` (`backend/app/notification/services/notification_send_service.py:126-128,205-207`).
- Policy-blocked attempts do not create notification rows. Preview returns `NotificationPreviewResponse(status="blocked")`; send/auto-send return `NotificationResult(status="blocked")` (`backend/app/notification/services/notification_send_service.py:139-153,219-232`, `backend/app/notification/services/notification_auto_send_service.py:138-146`).
- Client status `FROZEN` or `CLOSED` blocks all triggers except `client_missing_information` and `client_documents_request` (`backend/app/notification/services/notification_policy_service.py:19-24,61-69`).
- Trigger-specific policy checks enforce entity ownership/status rules before delivery:
  - `annual_report_client_reminder` requires the report to belong to the client, be in `PENDING_CLIENT`, and respect the 2-day cooldown after a `sent` reminder (`backend/app/notification/services/notification_policy_service.py:155-187`, `backend/app/notification/services/constants.py:3`).
  - `annual_report_documents_request` requires the report to belong to the client and be in `NOT_STARTED`, `COLLECTING_DOCS`, or `IN_PREPARATION` (`backend/app/notification/services/notification_policy_service.py:25-32,189-204`).
  - `vat_documents_reminder` requires the VAT work item to belong to the client, not be filed/canceled, use `due_date_effective`, and fall within the 14-day reminder window (`backend/app/notification/services/notification_policy_service.py:206-239`, `backend/docs/domain_decisions_v3.md:390`).
  - `payment_reminder` requires a charge belonging to the client in `ISSUED`; a recent sent reminder returns a warning unless `confirm_recent_duplicate=true` (`backend/app/notification/services/notification_policy_service.py:241-275`, `backend/app/notification/services/constants.py:4`).
  - `invoice_issued` requires a charge belonging to the client in `ISSUED` or `PAID` (`backend/app/notification/services/notification_policy_service.py:277-287`).
  - Signature triggers require a matching signature request, `PENDING_SIGNATURE`, a non-expired request, and an active `signing_token` (`backend/app/notification/services/notification_policy_service.py:289-306`).
- Template context is resolved from the trigger and entity. Binder triggers can add `binder_number`; annual-report triggers add `tax_year`; charge, VAT, and signature triggers inject entity-specific fields; client-level free-text triggers default `message` to an empty string in previews (`backend/app/notification/services/notification_context_resolver.py:28-109`).
- Recipient resolution differs by trigger: signature triggers use `SignatureRequest.signer_email`; all other triggers resolve the OWNER `Person` email from the client record. If no recipient is found, the service persists a `skipped` row with `recipient=null` and returns `status="skipped"` (`backend/app/notification/services/notification_send_service.py:321-372,417-422`, `backend/app/notification/services/notification_context_resolver.py:111-139`).
- Manual send validates the final trimmed subject/body after policy and template resolution. Empty values, oversize values, or visible `{placeholder}` tokens raise `NOTIFICATION.*` validation errors instead of creating a row (`backend/app/notification/services/notification_send_service.py:234-288`, `backend/app/notification/services/constants.py:6-8`).
- Manual send hashes the effective payload and reuses prior non-`pending` results for the same idempotency key within 24 hours. A prior `pending` row is treated as an interrupted attempt and does not short-circuit the retry (`backend/app/notification/services/notification_send_service.py:71-92,290-318`, `backend/app/notification/services/constants.py:6`).
- Both manual and automatic services create a `pending` row before delivery, call `NotificationDeliveryService.send()`, then mark it `sent` or `failed`. `NotificationDeliveryService` itself never persists notification state (`backend/app/notification/services/notification_send_service.py:374-415`, `backend/app/notification/services/notification_auto_send_service.py:193-224`, `backend/app/notification/services/notification_delivery_service.py:8-31`).
- Real outbound email is enabled only when `APP_ENV` is `staging` or `production` and `NOTIFICATIONS_ENABLED` is true. In other environments, the email channel is constructed disabled (`backend/app/notification/services/notification_send_service.py:102-110`, `backend/app/notification/services/notification_auto_send_service.py:82-90`).
- Automatic sends are internal-only. `NotificationAutoSendService` permits only `binder_ready_for_handover`, and `BinderLifecycleService.mark_ready_for_handover()` is the current caller that generates the idempotency key and passes binder context (`backend/app/notification/services/notification_auto_send_service.py:19,67-111`, `backend/app/binders/services/binder_lifecycle_service.py:167-179`).

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | Raised when |
|------|-------------|
| `NOTIFICATION.MISSING_IDEMPOTENCY_KEY` | Manual send is missing `X-Idempotency-Key`, or auto-send receives a blank idempotency key (`backend/app/notification/api/notifications.py:120-127`, `backend/app/notification/services/notification_auto_send_service.py:110-111`) |
| `NOTIFICATION.INVALID_IDEMPOTENCY_KEY` | Manual send header is present but not a valid UUID (`backend/app/notification/api/notifications.py:128-135`) |
| `NOTIFICATION.AUTO_SEND_TRIGGER_NOT_ALLOWED` | Internal auto-send is called with a trigger other than `binder_ready_for_handover` (`backend/app/notification/services/notification_auto_send_service.py:105-109`) |
| `NOTIFICATION.AUTO_ONLY_TRIGGER` | Manual preview/send tries to use `binder_ready_for_handover` (`backend/app/notification/services/notification_send_service.py:119-124,198-203`) |
| `NOTIFICATION.MISSING_ENTITY_ID` | A trigger that requires an entity id is called without one (`backend/app/notification/services/notification_send_service.py:121-124,200-203`) |
| `NOTIFICATION.EMPTY_SUBJECT` | Final send subject is blank after trimming (`backend/app/notification/services/notification_send_service.py:269-275`) |
| `NOTIFICATION.EMPTY_BODY` | Final send body is blank after trimming (`backend/app/notification/services/notification_send_service.py:269-276`) |
| `NOTIFICATION.SUBJECT_TOO_LONG` | Final send subject exceeds `SUBJECT_MAX_LENGTH` (`backend/app/notification/services/notification_send_service.py:277-281`) |
| `NOTIFICATION.BODY_TOO_LONG` | Final send body exceeds `BODY_MAX_LENGTH` (`backend/app/notification/services/notification_send_service.py:282-286`) |
| `NOTIFICATION.VISIBLE_PLACEHOLDER` | Final send subject/body still contains an unresolved `{placeholder}` token (`backend/app/notification/services/notification_send_service.py:272,287-288`) |
| `NOTIFICATION.TEMPLATE_ERROR` | Template rendering fails because the trigger template is missing, a required key is absent, or placeholders remain after render (`backend/app/notification/services/notification_template_renderer.py:12,33-60`) |

## Known issues

None currently tracked.

## Decisions (preserved)

Still-true decisions carried over from the historical notification README and shared backend docs, after checking current code:

1. **Blocked attempts are policy outcomes, not notification-history rows.** Current send services return `blocked` without persisting a notification record, while skipped/sent/failed attempts do persist rows (`docs/archive/notifications-legacy.md`, `backend/app/notification/services/notification_send_service.py:219-232,343-415`, `backend/app/notification/services/notification_auto_send_service.py:145-224`).
2. **Notification history stores rendered content snapshots.** `content_snapshot` and `subject_snapshot` are persisted on every saved attempt so later template changes do not rewrite history (`docs/archive/notifications-legacy.md`, `backend/app/notification/models/notification.py:120-123`, `backend/app/notification/repositories/notification_repository.py:22-61`).
3. **The current automatic path is limited to binder handover.** The README described only a limited automatic path, and code still restricts auto-send to `binder_ready_for_handover` initiated from binder lifecycle (`docs/archive/notifications-legacy.md`, `backend/app/notification/services/notification_auto_send_service.py:19,67-111`, `backend/app/binders/services/binder_lifecycle_service.py:167-179`).
4. **VAT reminder policy must use `due_date_effective`.** The shared domain decisions doc explicitly says notification policy should use `due_date_effective` where relevant, and the VAT reminder rule does exactly that (`backend/docs/domain_decisions_v3.md:390`, `backend/app/notification/services/notification_policy_service.py:228-237`).

## Future / planned

- Additional delivery channels are still only planned at the notification-domain level. The model enum includes `whatsapp`, but both current send services construct and use only the email channel (`docs/archive/notifications-legacy.md`, `backend/app/notification/models/notification.py:27-30`, `backend/app/notification/services/notification_send_service.py:102-110,290`, `backend/app/notification/services/notification_auto_send_service.py:82-90,163-217`).
- Automatic sending beyond binder handover is not implemented. `NotificationAutoSendService` explicitly whitelists only `binder_ready_for_handover` (`docs/archive/notifications-legacy.md`, `backend/app/notification/services/notification_auto_send_service.py:19,105-109`).

## Historical notes

Historical notification-domain material from the old module README was archived to `docs/archive/notifications-legacy.md`. The original file at `backend/app/notification/README.md` now serves only as a pointer.
