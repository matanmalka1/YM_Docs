## Scope
This file owns only:
- Canonical current-state documentation for the signature-requests domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Signature Requests

Manages the full lifecycle of digital signature requests sent to clients for legally-binding approval of documents. Built under Israeli Electronic Signature Law 5761-2001: every request anchors to a `client_record_id`, carries a one-time signing token, captures a content hash for tamper detection, and records every lifecycle transition in an append-only audit trail.

Last verified against code + backend/openapi.json: 2026-06-11 for `updated_at` on the request row (#46); full-domain verification remains 2026-06-05.

## Endpoints

### Advisor routes (JWT required — roles: `ADVISOR`, `SECRETARY`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/v1/signature-requests | Create and immediately send a signature request |
| GET | /api/v1/signature-requests/pending | List all `pending_signature` requests (paginated) |
| GET | /api/v1/signature-requests/{request_id} | Get request details + embedded audit trail |
| POST | /api/v1/clients/{client_record_id}/signature-requests/{request_id}/cancel | Cancel a pending request within its owning client scope |
| GET | /api/v1/clients/{client_record_id}/signature-requests | List all requests for a client record (paginated, filterable by status) |

### Public signer routes (no JWT — token-based)

| Method | Path | Purpose |
|--------|------|---------|
| GET | /sign/{token} | Signer views the request; records `viewed` audit event |
| POST | /sign/{token}/approve | Signer approves; transitions `pending_signature → signed` |
| POST | /sign/{token}/decline | Signer declines; transitions `pending_signature → declined` |

All paths confirmed in `backend/openapi.json`.

## Model & fields

### `SignatureRequest` — table `signature_requests`

Source: `backend/app/signature_requests/models/signature_request.py:56`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | autoincrement |
| `client_record_id` | FK → `client_records.id` | no | primary anchor; indexed |
| `business_id` | FK → `businesses.id` | yes | optional scope context; indexed |
| `created_by` | FK → `users.id` | no | advisor who created |
| `annual_report_id` | FK → `annual_reports.id` | yes | cross-domain link; indexed |
| `document_id` | FK → `permanent_documents.id` | yes | cross-domain link |
| `request_type` | pg_enum(`SignatureRequestType`) | no | |
| `title` | String | no | |
| `description` | Text | yes | |
| `content_hash` | String | yes | SHA-256 of provided content |
| `storage_key` | String | yes | S3/R2 key of original document |
| `signer_name` | String | no | |
| `signer_email` | String | yes | may be filled from business profile |
| `signer_phone` | String | yes | may be filled from business profile |
| `status` | pg_enum(`SignatureRequestStatus`) | no | default `pending_signature` |
| `signing_token` | String | yes | unique one-time URL-safe token; cleared on terminal state |
| `created_at` | datetime | no | UTC |
| `updated_at` | datetime | yes | `onupdate=utcnow`; set on real row mutation (send/sign/decline/cancel/expire/soft-delete); NULL until first update — never faked from `created_at` (#46). Distinct from the append-only audit trail. |
| `sent_at` | datetime | yes | set at creation time |
| `expires_at` | datetime | yes | `created_at + expiry_days` |
| `expiry_days` | int | no | default 14, max 90 |
| `signed_at` | datetime | yes | set when signed |
| `declined_at` | datetime | yes | set when declined |
| `canceled_at` | datetime | yes | set when canceled |
| `canceled_by` | FK → `users.id` | yes | advisor who canceled |
| `signer_ip_address` | String | yes | captured at signing/declining; excluded from API response (PII) |
| `signer_user_agent` | String | yes | captured at signing/declining |
| `decline_reason` | Text | yes | optional signer-provided reason |
| `signed_document_key` | String | yes | S3/R2 key for countersigned PDF |
| `deleted_at` | datetime | yes | soft delete |
| `deleted_by` | FK → `users.id` | yes | soft delete actor |

Indexes: `(client_record_id)`, `(business_id)`, `(annual_report_id)`, `(status)`, partial `(status, sent_at) WHERE deleted_at IS NULL`.

### `SignatureAuditEvent` — table `signature_audit_events`

Source: `backend/app/signature_requests/models/signature_request.py:157`

Append-only — no soft delete, no `updated_at`.

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| `id` | int PK | no | |
| `signature_request_id` | FK → `signature_requests.id` | no | indexed |
| `event_type` | String | no | free-form; not an enum (see below) |
| `actor_type` | String | no | `advisor` / `signer` / `system` |
| `actor_id` | int | yes | advisor user ID if applicable |
| `actor_name` | String | yes | |
| `ip_address` | String | yes | |
| `user_agent` | String | yes | |
| `notes` | Text | yes | Hebrew narrative note |
| `occurred_at` | datetime | no | UTC |

Index: `(occurred_at)`.

## Enums / statuses

### `SignatureRequestStatus`

Source: `backend/app/signature_requests/models/signature_request.py:40`

| Value | Meaning |
|-------|---------|
| `pending_signature` | Awaiting signer action |
| `signed` | Signer approved |
| `declined` | Signer declined |
| `expired` | Past `expires_at` with no action |
| `canceled` | Canceled by advisor or system |

State machine: `pending_signature → signed | declined | expired | canceled`. All transitions are terminal.

### `SignatureRequestType`

Source: `backend/app/signature_requests/models/signature_request.py:48`

| Value | Hebrew label |
|-------|-------------|
| `engagement_agreement` | הסכם התקשרות |
| `annual_report_approval` | אישור דוח שנתי |
| `power_of_attorney` | ייפוי כוח |
| `vat_return_approval` | אישור דוח מע"מ |
| `custom` | חתימה כללית |

### `event_type` values (audit trail)

`event_type` and `actor_type` are plain `String` columns — intentionally not enum, so new event types require no migration. Known values: `created`, `sent`, `viewed`, `signed`, `declined`, `canceled`, `expired`, `annual_report_signed`.

Source: `backend/app/signature_requests/models/signature_request.py:24`, `services/messages.py`.

## Domain rules & invariants

Source: `backend/app/signature_requests/services/`

1. **`client_record_id` always required.** Every request must belong to a client record. Checked at creation; raises `CLIENT_RECORD.NOT_FOUND` if missing. (`create_request.py:57-59`)

2. **`business_id` ownership validated on create.** When `business_id` is provided, it must belong to the `legal_entity_id` of the client record. Delegates to `assert_business_belongs_to_legal_entity`. (`create_request.py:62-68`)

3. **Signer contact fallback.** If `signer_email` or `signer_phone` is absent and `business_id` is set, values are pulled from the business profile via `BusinessContactService`. (`create_request.py:69-75`)

4. **`request_type` validated against enum.** Invalid values raise `SIGNATURE_REQUEST.INVALID_TYPE`. (`create_request.py:77-85`)

5. **Content hash computed server-side.** If caller supplies `content_to_hash`, the service computes `SHA-256` and stores it. Callers never write `content_hash` directly. (`create_request.py:88-90`)

6. **Token is unguessable and one-time.** Generated with `secrets.token_urlsafe(32)`. Cleared (`NULL`) immediately upon any terminal transition (signed / declined / canceled / expired). (`create_request.py:109`, `signer_actions.py:77-85`, `admin_actions.py:38-43`)

7. **Row-level lock on state transitions.** `sign_request`, `decline_request`, and `cancel_request` all use `SELECT … FOR UPDATE` via token-scoped or client-and-request-scoped repository lookups to prevent race conditions.

8. **Signer actions require `pending_signature` status.** `assert_pending` raises `SIGNATURE_REQUEST.INVALID_STATUS` if not pending. (`signature_request_validations.py:57-67`)

9. **Runtime expiry detection.** If `expires_at` has passed at the time of a signer action, the service atomically transitions to `EXPIRED`, clears the token, appends an `expired` audit event, and raises `SIGNATURE_REQUEST.EXPIRED`. (`signer_actions.py:28-37`)

10. **Cancel is client-scoped and pending-only.** Cancellation requires both `client_record_id` and `request_id`; the repository resolves only a matching `pending_signature` row under a row lock. A mismatched client, missing request, or terminal request raises `SIGNATURE_REQUEST.NOT_FOUND`.

11. **Batch expiry job.** `expire_overdue_requests()` scans all `pending_signature` rows past `expires_at`, transitions them to `EXPIRED`, and appends audit events. Intended for a periodic scheduler. (`admin_actions.py:59-79`)

12. **Annual-report auto-advance on sign.** When a signed request has `annual_report_id` set and `request_type = annual_report_approval`, `SignatureRequestService.sign_request` attempts a best-effort `pending_client → submitted` transition and sets `client_approved_at`. The call is wrapped in `try/except` — failure is logged, not surfaced to the signer. (`signature_request_service.py:84-108`)

13. **Audit trail is append-only.** `SignatureAuditEvent` has no `updated_at` and no soft-delete. Events for `created` and `sent` are always appended together at creation. (`create_request.py:115-130`, model docstring)

14. **`signer_ip_address` excluded from API responses.** Intentionally omitted from `SignatureRequestResponse` schema to avoid PII exposure. Available only in the audit trail. (`schemas/signature_request.py:55`)

15. **List by client returns all statuses by default.** Optional `status` query param filters; invalid values raise `SIGNATURE_REQUEST.INVALID_STATUS`. (`signature_request_service.py:157-167`)

## Error codes

Source: `backend/app/signature_requests/services/`, `docs/architecture/error-codes.md`.

| Code | HTTP | When raised |
|------|------|-------------|
| `SIGNATURE_REQUEST.NOT_FOUND` | 404 | Request ID not found |
| `SIGNATURE_REQUEST.INVALID_TYPE` | 400 | `request_type` not in enum |
| `SIGNATURE_REQUEST.INVALID_STATUS` | 400 | Signer action on terminal request or invalid list filter |
| `SIGNATURE_REQUEST.TOKEN_INVALID` | 400 | Token not found or already cleared |
| `SIGNATURE_REQUEST.EXPIRED` | 400/410 | Signer action attempted after `expires_at` |
| `CLIENT_RECORD.NOT_FOUND` | 404 | `client_record_id` not found at create or list |
| `BUSINESS.NOT_FOUND` | 404 | `business_id` not found at create |

All codes follow `DOMAIN.REASON` format. Registry: `docs/architecture/error-codes.md`.

## Known issues

No open known issues.

## Resolved issues

- **F-029** (2026-06-05): Detail lookup raised the generic `not_found` code via raw `HTTPException`. Fixed: now raises `SIGNATURE_REQUEST.NOT_FOUND` through `NotFoundError`.
- **F-030** (2026-06-05): Cancellation route was bare and used an unscoped lookup. Fixed: `POST /api/v1/clients/{client_record_id}/signature-requests/{request_id}/cancel` with locked client-and-request-scoped pending lookup.
- **F-031** (2026-06-05): Closed as stale; reported orphaned bytecode absent; `__pycache__/` is ignored.

## Decisions (preserved)

From `backend/app/signature_requests/README.md` (audited 2026-03-22) and model docstring:

1. **`client_record_id` as primary anchor.** Signature requests are anchored to the legal entity record, not to a specific business, because the signer is typically the legal representative of the entity. `business_id` is optional context.

2. **`signing_token` is cleared after terminal state.** Once a request reaches `signed`, `declined`, `canceled`, or `expired`, the token is set to `NULL`. This prevents replay of a previously-used token.

3. **`event_type` and `actor_type` are `String`, not enum.** Allows the audit trail to gain new event types without schema migrations. This is a deliberate tradeoff: type safety is sacrificed for migration-free extensibility.

4. **`SignatureAuditEvent` has no soft delete and no `updated_at`.** The audit trail is legally required to be immutable. Adding delete or update columns would undermine its evidentiary value.

5. **Content hash is SHA-256 of caller-supplied text.** The service does not hash the stored file — it hashes `content_to_hash` provided at creation time. This allows hashing of a canonical serialization without streaming from S3/R2.

6. **Auto-advance of annual report is best-effort.** The `_auto_advance_annual_report` call is inside a broad `try/except Exception` and logs on failure. The signer's action always succeeds regardless of the annual report transition outcome.

7. **Israeli Electronic Signature Law 5761-2001 compliance intent.** The model captures: who was asked (signer identity), when asked (sent_at), what they approved (content_hash), and how they confirmed (signed_at + signer_ip_address + signer_user_agent). These four elements satisfy the audit-trail requirement.

## Future / planned

- **Countersigned PDF storage** (`signed_document_key`): The column `signed_document_key` exists in the model for a countersigned PDF to be stored in S3/R2 after approval, but no service currently writes to it. The feature is scaffolded, not implemented.

- **Batch expiry scheduler**: `expire_overdue_requests()` is implemented in `admin_actions.py` but there is no evidence of a background job or cron that calls it (cf. F-016 pattern in reminders). Expiry at signer-action time works; batch sweep relies on a scheduler being wired up.

- **Notification delivery**: No email/SMS sending is implemented in the signature request creation flow. The response returns `signing_url_hint` for the advisor to distribute manually.

## Historical notes

No legacy files found under `backend/docs/` for this domain. No archive created.
