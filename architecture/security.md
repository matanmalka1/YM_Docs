## Scope
This file owns only:
- Authorization, auth token policy, IDOR prevention, sensitive operations, ownership checks, and browser auth storage rules.

This file must not contain:
- General backend layering, product permissions by module, or deployment secrets inventory unless directly security-related.

Source of truth: mandatory

# Security

- Backend authorization is the source of truth.
- Endpoint-level role checks belong in routers.
- Fine-grained permission and ownership checks belong in services.
- Sensitive data must be scoped by business/client ownership or fetched and checked before return.
- New sensitive endpoints must avoid IDOR by construction.
- Sensitive write operations, imports, bulk actions, and background triggers must require idempotency where applicable.
- Browser SPA auth should use short-lived access tokens in memory and long-lived refresh tokens in HttpOnly cookies.
- Do not store access tokens or refresh tokens in `localStorage` or `sessionStorage` unless explicitly requested.
- On app bootstrap, call refresh, store the returned access token in memory, then fetch the current user.
- API clients should retry an authenticated request once after successful refresh.
- For deployed SPA plus separate API projects, prefer same-origin API rewrites/proxies for cookie auth.
- Refresh cookies should be scoped narrowly with `HttpOnly`, `Secure` in production, explicit `SameSite`, and an auth-specific `Path` when practical.
