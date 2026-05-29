## Scope
This file owns only:
- Authorization, auth token policy, IDOR prevention, sensitive operations, ownership checks, and browser auth storage rules.

This file must not contain:
- General backend layering, product permissions by module, or deployment secrets inventory unless directly security-related.

Source of truth: mandatory

# Security

- Backend authorization is the source of truth.
- Access tokens are JWT HS256 bearer tokens and must be short-lived.
- The default access-token lifetime is 15 minutes.
- Refresh tokens are JWT HS256 tokens stored in HttpOnly cookies scoped to `/api/v1/auth`.
- The default refresh-token lifetime is 7 days.
- User tokens are invalidated with the `token_version` column; JWT payloads include `tv`, and auth dependencies must reject tokens whose version does not match the database value.
- Logout must invalidate outstanding tokens by incrementing `token_version` and clearing the refresh cookie.
- Endpoint-level role checks must live in routers.
- Endpoint-level role checks use `require_role(...)`.
- Fine-grained permission and ownership checks must live in services.
- Route signatures should use `CurrentUser` for authenticated users; it resolves to an `AuthSubject` projection, not the full User ORM model.
- Sensitive data must be scoped by business/client ownership or fetched and checked before return.
- Sensitive query scoping must use `business_id`, `client_record_id`, or an equivalent ownership key.
- New sensitive endpoints must avoid IDOR by construction.
- Bulk actions, imports, external delivery, payment-like operations, and automatic/background triggers must require idempotency unless the owning doc explicitly excludes them.
- Idempotent operations should use `IdempotencyService` from `app/infrastructure/idempotency/` to store request fingerprints and return cached responses on replay.
- Browser SPA auth must use short-lived access tokens in memory and long-lived refresh tokens in HttpOnly cookies unless the user explicitly chooses another tradeoff.
- Do not store access tokens or refresh tokens in `localStorage` or `sessionStorage` unless explicitly requested.
- On app bootstrap, call refresh, store the returned access token in memory, then fetch the current user.
- API clients must retry an authenticated request at most once after successful refresh.
- Deployed SPA plus separate API projects must use same-origin API rewrites/proxies for cookie auth unless the user explicitly accepts cross-site cookie risk.
- Refresh cookies must use `HttpOnly`, `Secure` in production, explicit `SameSite`, and the narrowest practical `Path`.
- Authentication must audit login attempts and logout events with reason codes for failed login attempts.
- Authentication audit records are stored in `user_audit_logs`; failed login reason codes include `user_not_found`, `inactive_user`, and `invalid_password`.
- The login endpoint must be rate-limited by `AUTH_LOGIN_RATE_LIMIT`; the default outside tests is `5/minute`.
