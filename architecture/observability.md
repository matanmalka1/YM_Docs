## Scope
This file owns only:
- Logging, request IDs, error logging, request summaries, Sentry policy, and SQL logging in development.

This file must not contain:
- General testing rules, product audit history rules, or domain timeline behavior.

Source of truth: mandatory

# Observability

- Every request should have a request ID available to logs and error reports.
- Logs should include enough context to debug failures without leaking sensitive data.
- Error logs should preserve exception context for server-side debugging.
- User-facing errors must not expose stack traces.
- Request summary logs should be useful for latency and failure investigation.
- SQL logging is a development/debugging tool and must not leak sensitive production data.
- Sentry or equivalent error reporting should receive server-side exceptions with request context when configured.
- Product audit history and domain timelines are product behavior, not observability.
