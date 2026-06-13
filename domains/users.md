## Scope
This file owns only:
- Canonical current-state documentation for the users domain.

This file must not contain:
- Architecture/API rules (link to docs/architecture/*).
- Other domains' behavior.

Source of truth: mandatory

# Users

The users domain manages the full lifecycle of system operator accounts: authentication (login/logout/token refresh), self-service password reset, advisor-only user administration (create/list/get/update/activate/deactivate/reset password), and structured audit logging of all auth and management actions. It also provides the JWT validation dependency used by every other domain to identify and authorize the current request caller.

Last verified against code + backend/openapi.json: 2026-05-29.

## Endpoints

All paths exist in `backend/openapi.json`.

### Auth (public + authenticated)

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | /api/v1/auth/login | Public | Authenticate; returns access token + sets HttpOnly refresh cookie |
| POST | /api/v1/auth/refresh | Cookie | Exchange refresh cookie for a new access token |
| GET | /api/v1/auth/me | Bearer | Return current user's id, full_name, role, email |
| POST | /api/v1/auth/logout | Cookie | Bump token_version (invalidate all tokens); clear refresh cookie |
| POST | /api/v1/auth/forgot-password | Public | Request password reset email (always returns generic message) |
| POST | /api/v1/auth/reset-password | Public | Consume one-time token; set new password; bump token_version |

### User management (ADVISOR only)

| Method | Path | Purpose |
|--------|------|---------|
| POST | /api/v1/users | Create user |
| GET | /api/v1/users | List users (paginated; filters: is_active, search) |
| GET | /api/v1/users/audit-logs | List audit log entries (paginated; filters: action, target_user_id, actor_user_id, email, created_after, created_before) |
| GET | /api/v1/users/{user_id} | Get single user |
| PATCH | /api/v1/users/{user_id} | Partial update (full_name, phone, role, email) |
| POST | /api/v1/users/{user_id}/activate | Activate user |
| POST | /api/v1/users/{user_id}/deactivate | Deactivate user + bump token_version |
| POST | /api/v1/users/{user_id}/reset-password | Set new password + bump token_version |

## Model & fields

### `User` — table `users`
Source: `backend/app/users/models/user.py:19`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | No | autoincrement |
| full_name | str | No | |
| email | str | No | unique, indexed |
| phone | str | Yes | |
| password_hash | str(60) | No | bcrypt hash |
| role | UserRole enum | No | pg enum |
| is_active | bool | No | default true |
| token_version | int | No | default 0; bumped on logout/deactivate/password reset |
| created_at | datetime | No | utcnow |
| last_login_at | datetime | Yes | updated on each successful login |

### `UserAuditLog` — table `user_audit_logs`
Source: `backend/app/users/models/user_audit_log.py:30`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | No | |
| action | AuditAction enum | No | indexed |
| actor_user_id | int FK→users.id | Yes | indexed; null for unauthenticated events |
| target_user_id | int FK→users.id | Yes | indexed |
| email | str | Yes | indexed; present for login attempts |
| status | AuditStatus enum | No | indexed |
| reason | str | Yes | machine-readable failure reason |
| metadata_json | text | Yes | JSON blob (e.g. updated_fields list) |
| created_at | datetime | No | indexed |

### `PasswordResetToken` — table `password_reset_tokens`
Source: `backend/app/users/models/password_reset_token.py:12`

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | int PK | No | |
| user_id | int FK→users.id | No | indexed |
| token_hash | str(64) | No | SHA-256 of raw token; unique constraint |
| expires_at | datetime | No | indexed |
| used_at | datetime | Yes | set on consumption |
| created_at | datetime | No | |
| requested_ip | str(45) | Yes | |
| user_agent | str(512) | Yes | |

## Enums / statuses

### `UserRole` — `backend/app/users/models/user.py:14`
| Value | Description |
|-------|-------------|
| `advisor` | Full administrative access; can manage users |
| `secretary` | Restricted access; cannot manage users |

### `AuditAction` — `backend/app/users/models/user_audit_log.py:14`
| Value |
|-------|
| `login_success` |
| `login_failure` |
| `logout` |
| `user_created` |
| `user_updated` |
| `user_activated` |
| `user_deactivated` |
| `password_reset` |

### `AuditStatus` — `backend/app/users/models/user_audit_log.py:25`
| Value |
|-------|
| `success` |
| `failure` |

## Domain rules & invariants

**Role gating**
- All user-management endpoints (`/api/v1/users/**`) are ADVISOR-only at the router level via `Depends(require_role(UserRole.ADVISOR))`. Source: `backend/app/users/api/users.py:14-18`, `backend/app/users/api/users_audit.py:14-18`.
- `ensure_advisor()` also enforces this in every management service method as a secondary check. Source: `backend/app/users/services/user_management_policies.py:11-13`.

**Authentication**
- Email is normalized (strip + lowercase) before lookup at login and forgot-password. Source: `backend/app/users/services/auth_service.py:58`, `backend/app/users/services/password_reset_service.py:51`.
- Inactive users cannot log in; login returns `None` and logs `LOGIN_FAILURE` with reason `inactive_user`. Source: `backend/app/users/services/auth_service.py:71-79`.
- Access token payload: `sub` (user id str), `email`, `role`, `tv` (token_version), `type`, `iat`, `exp`. Source: `backend/app/users/services/token_service.py:19-28`.
- Per-request auth re-checks `is_active` and `token_version` against DB on every request. Source: `backend/app/users/api/deps.py:47-58`.

**Token invalidation via `token_version`**
- Logout bumps `token_version` (via `bump_token_version`). All previously issued JWTs for that user are immediately rejected even before expiry. Source: `backend/app/users/services/auth_service.py:117`.
- Deactivate bumps `token_version` + sets `is_active=False` atomically. Source: `backend/app/users/repositories/user_repository.py:164-172`.
- Admin password reset bumps `token_version` + updates `password_hash` atomically. Source: `backend/app/users/repositories/user_repository.py:175-183`.
- Self-service password reset bumps `token_version` + updates `password_hash` inline in service. Source: `backend/app/users/services/password_reset_service.py:109-111`.

**Self-deactivation guard**
- A user cannot deactivate their own account. Service raises `USER.FORBIDDEN` when `actor_user_id == target_user_id`. Source: `backend/app/users/services/user_management_service.py:152-153`.

**Password rules**
- 8–128 characters; must contain at least one uppercase letter (`[A-Z]`), one lowercase letter (`[a-z]`), and one special character (non-alphanumeric). Enforced via `validate_password()`. Source: `backend/app/users/services/user_management_policies.py:6-27`.

**Immutable user fields**
- `id`, `token_version`, `created_at`, `last_login_at`, `is_active` cannot be set via PATCH. Attempt raises `USER.INVALID_UPDATE`. Source: `backend/app/users/services/user_management_service.py:14-20`.

**Email uniqueness**
- Email unique at DB level (unique index). Service pre-checks before insert/update and raises `USER.CONFLICT` on conflict. Source: `backend/app/users/services/user_management_service.py:55-56`, `103-108`.

**Password reset token lifecycle**
- Raw token is `secrets.token_urlsafe(32)`; only the SHA-256 hash is stored. Source: `backend/app/users/services/password_reset_service.py:58`.
- All previous unused tokens for the same user are invalidated when a new request is made. Source: `backend/app/users/services/password_reset_service.py:56`.
- Token is marked used before the password update; if marking fails, the whole operation is rejected. Source: `backend/app/users/services/password_reset_service.py:106-107`.
- `forgot-password` always returns the same generic message regardless of whether the email exists — prevents user enumeration. Source: `backend/app/users/services/password_reset_service.py:53-54`.

**Refresh cookie**
- `httponly=True`; `secure=True` in production, `False` otherwise; `samesite` and `path` from config. Cookie scoped to `/api/v1/auth`. Source: `backend/app/users/api/auth_cookies.py`, `backend/app/users/api/constants.py`.

**Rate limiting**
- Login: rate-limited by email key (limit from `settings.AUTH_LOGIN_RATE_LIMIT`). Source: `backend/app/users/api/auth.py:22`.
- Forgot-password: `3/hour` per email. Source: `backend/app/users/api/password_reset.py:17`.
- Reset-password: `10/minute`. Source: `backend/app/users/api/password_reset.py:34`.

**Audit logging**
- All auth events (login success/failure, logout) and all management events (create, update, activate, deactivate, password reset) are written to `user_audit_logs`. Login failures include a machine-readable `reason` field (`user_not_found`, `inactive_user`, `invalid_password`).

## Error codes

Registry: `docs/architecture/error-codes.md`.

| Code | HTTP | Raised when |
|------|------|-------------|
| `USER.NOT_FOUND` | 404 | user_id does not exist |
| `USER.CONFLICT` | 409 | email already in use |
| `USER.FORBIDDEN` | 403 | non-advisor attempts management action; advisor attempts self-deactivation |
| `USER.INVALID_PASSWORD` | 400 | password fails length or complexity rules |
| `USER.INVALID_UPDATE` | 400 | attempt to set an immutable field |
| `USER.NO_FIELDS_PROVIDED` | 400 | PATCH body resolves to zero updateable fields |
| `AUTH.INVALID_REFRESH_TOKEN` | 401 | refresh token absent, expired, wrong type, or token_version mismatch |
| `AUTH.INVALID_PASSWORD_RESET_TOKEN` | 400 | reset token absent, expired, already used, or user inactive |

All codes follow the `DOMAIN.REASON` format. Source: `backend/app/users/services/auth_service.py:32`, `backend/app/users/services/password_reset_service.py:100,104,107`, `backend/app/users/services/user_management_service.py:31,56,107,115,119,153`, `backend/app/users/services/user_management_policies.py:13,18,20,22,26`.

## Known issues

No open known issues.

## Decisions (preserved)

- **Token invalidation via `token_version` (not token revocation list).** Chosen to avoid the need for a server-side revocation store. The version integer is embedded in every JWT payload and re-checked against the DB on every request. Any single bump invalidates all previously issued tokens for that user simultaneously. This is intentional for logout, deactivation, and password reset scenarios.

- **Dual-path logout.** `POST /api/v1/auth/logout` resolves the user from the refresh cookie (if present) and bumps token_version. If no cookie is present, it still clears the cookie header on the response. This allows logout to succeed even in degraded cookie states.

- **ADVISOR-only user management.** SECRETARY role has no user-management capability by design. Role guard is enforced at both router-level dependency and service level.

- **Forgot-password returns uniform message.** Always returns the same response string regardless of whether the email corresponds to an existing/active user, preventing enumeration attacks.

- **Password reset token hash-only storage.** The raw token is never persisted; only the SHA-256 hash is stored. The raw token travels only in the reset URL sent by email.

## Resolved issues

- **F-041 / F-users-001** (2026-06-05): `GET /api/v1/users/audit-logs` accepted reversed date ranges and silently returned empty results. Fixed: inline guard in `users_audit.py` raises `AppError("USER.INVALID_DATE_RANGE", 400)` when `created_after > created_before`.

## Future / planned

- **`JWT_SECRET` rotation strategy.** No documented rotation/revocation procedure. The module README (last audited 2026-05-26) flags this as a known gap.
- **Concurrent duplicate-email handling.** Duplicate email is caught by a service-level pre-check before insert, but a concurrent create race could surface as a raw DB unique-constraint error rather than a normalized `USER.CONFLICT` response. Normalization at the global exception handler is not confirmed.

## Historical notes

No legacy files exist under `backend/docs` for the users domain. The in-module file `backend/app/users/README.md` is the current source-side reference and was used as a cross-check during authoring; it is not a legacy file and has not been replaced.
