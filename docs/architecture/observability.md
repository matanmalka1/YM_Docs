## Scope
This file owns only:
- Logging, request IDs, error logging, request summaries, Sentry policy, and SQL logging in development.

This file must not contain:
- General testing rules, product audit history rules, or domain timeline behavior.

Source of truth: mandatory

# Observability

- Every request must have a request ID available to logs and error reports.
- Logs must include enough context to debug failures without leaking sensitive data.
- Error logs must preserve exception context for server-side debugging.
- User-facing errors must not expose stack traces.
- Request summary logs must include enough information to investigate latency and failures.
- SQL logging must be limited to development/debugging contexts.
- SQL logs must not leak sensitive production data.
- Sentry or equivalent error reporting must receive unhandled server-side exceptions with request context when configured.
- Product audit history and domain timelines are product behavior, not observability.
