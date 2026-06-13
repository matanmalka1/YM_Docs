## Scope
This file owns only:
- A central registry of application error-code namespaces and the shared status mapping.

This file must not contain:
- The binding API error-envelope rules (those live in `docs/architecture/api-contracts.md`).
- Per-endpoint behavior or domain business rules.

Source of truth: reference

# Error Codes

This file is non-normative. The binding error-envelope and `error.code` rules live in `docs/architecture/api-contracts.md`. The authoritative list of codes is the backend service code; the `AppError` base classes live in `backend/app/core/exceptions.py`. Find every code in use with:

```bash
grep -rhoE '"[A-Z][A-Z_]+\.[A-Z][A-Z_]+"' --include='*.py' backend | sort -u
```

## Format

Codes use the `DOMAIN.REASON` pattern, e.g. `BINDER.NOT_FOUND`, `AUTH.INVALID_REFRESH_TOKEN`. `DOMAIN` is a stable namespace; `REASON` is upper snake case. Clients must match on `error.code`, never on translated message text.

## Status mapping

`AppError` subclasses map a code to an HTTP status (`backend/app/core/exceptions.py`):

| Class | Status |
|-------|--------|
| `AppError` (base) | 400 (default) |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `ForbiddenError` | 403 |

Transport/framework errors map by status in `exception_handlers.py`: 401 unauthorized, 405 method not allowed, 422 validation error, 429 rate limited, 500 internal server error.

## Domain namespaces

Each domain owns its `DOMAIN.*` codes. Registered prefixes:

| Namespace | Area |
|-----------|------|
| `ADVANCE_PAYMENT` | advance payments / מקדמות |
| `ANNUAL_REPORT` | annual reports / דוח שנתי |
| `AUDIT` | audit log entries |
| `AUTH` | authentication / tokens |
| `AUTHORITY_CONTACT` | authority contacts |
| `BINDER` | binders / קלסרים |
| `BINDER_INTAKE` | binder intake events / אירועי קבלה |
| `BUSINESS` | businesses / עוסקים |
| `BUSINESS_ACTIVITY` | business activity records |
| `CHARGE` | charges |
| `CLIENT` | clients / לקוחות |
| `CLIENT_RECORD` | client records |
| `CORRESPONDENCE` | correspondence |
| `DOCUMENT` | documents / uploads |
| `DOMAIN` | generic domain fallback |
| `INVOICE` | invoices / חשבוניות |
| `NOTE` | notes |
| `NOTIFICATION` | notifications / outbound delivery |
| `PERMANENT_DOCUMENTS` | permanent documents |
| `REMINDER` | reminders |
| `SIGNATURE_REQUEST` | signature requests / חתימות |
| `TASK` | tasks |
| `TAX_CALENDAR` | tax calendar / obligations |
| `TAX_ENGINE` | tax calculation engine |
| `TIMELINE` | client timeline |
| `USER` | users / accounts |
| `VAT` | VAT reports / מע״מ |

## Adding codes

### `BINDER_INTAKE` codes (selected)

| Code | HTTP | When raised |
|------|------|-------------|
| `BINDER_INTAKE.NOT_FOUND` | 404 | Intake ID does not exist, or `binder_id` path param does not match the intake's owning binder |

### `ANNUAL_REPORT` codes (selected)

| Code | HTTP | When raised |
|------|------|-------------|
| `ANNUAL_REPORT.AUDIT_ACTOR_REQUIRED` | 400 | VAT auto-populate service call omitted the actor id required for financial mutation audit |
| `ANNUAL_REPORT.SIGNER_NAME_MISSING` | 400 | `→ PENDING_CLIENT`: neither `Person.full_name` nor `LegalEntity.official_name` is populated for the client |

---

- Reuse an existing namespace for that domain; do not invent a second prefix for the same concept.
- Use a consistent reason verb across domains where it fits: `NOT_FOUND`, `CONFLICT`, `INVALID_STATUS`, `FORBIDDEN`.
- Raise an `AppError` subclass from the service layer so the status maps correctly; do not hardcode `HTTPException` for domain errors (see `docs/architecture/security.md` and `docs/adr/0002-router-service-repository.md`).
- A new namespace should be added to the table above in the same change.
