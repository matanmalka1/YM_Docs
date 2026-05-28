## Scope
This file owns only:
- Authorization, auth token policy, IDOR prevention, sensitive operations, ownership checks, and browser auth storage rules.

This file must not contain:
- General backend layering, product permissions by module, or deployment secrets inventory unless directly security-related.

Source of truth: mandatory

# Security

- Backend authorization is the source of truth.
- Endpoint-level role checks must live in routers.
- Fine-grained permission and ownership checks must live in services.
- Sensitive data must be scoped by business/client ownership or fetched and checked before return.
- New sensitive endpoints must avoid IDOR by construction.
- Bulk actions, imports, external delivery, payment-like operations, and automatic/background triggers must require idempotency unless the owning doc explicitly excludes them.
- Browser SPA auth must use short-lived access tokens in memory and long-lived refresh tokens in HttpOnly cookies unless the user explicitly chooses another tradeoff.
- Do not store access tokens or refresh tokens in `localStorage` or `sessionStorage` unless explicitly requested.
- On app bootstrap, call refresh, store the returned access token in memory, then fetch the current user.
- API clients must retry an authenticated request at most once after successful refresh.
- Deployed SPA plus separate API projects must use same-origin API rewrites/proxies for cookie auth unless the user explicitly accepts cross-site cookie risk.
- Refresh cookies must use `HttpOnly`, `Secure` in production, explicit `SameSite`, and the narrowest practical `Path`.
