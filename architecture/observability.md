## Scope
This file owns only:
- Logging, request IDs, error logging, request summaries, Sentry policy, and SQL logging in development.

This file must not contain:
- General testing rules, product audit history rules, or domain timeline behavior.

Source of truth: mandatory

# Observability

- Every request must have a request ID available to logs and error reports.
- Request IDs are generated or propagated from `X-Request-ID`, stored in request context, included in request logs, and echoed in the `X-Request-ID` response header.
- Logs must include enough context to debug failures without leaking sensitive data.
- Development and test logs use text format; staging and production logs use JSON format.
- `LOG_FORMAT=json` is required in staging and production.
- Backend logs go to stdout through a single stream handler.
- Backend modules should create loggers with `get_logger(__name__)` from `app/core/logging_config.py`.
- Backend logging is initialized once at startup with `setup_logging(level=settings.LOG_LEVEL, log_format=settings.LOG_FORMAT)`.
- Uvicorn access logs are disabled because request summary logs are the canonical access log.
- Error logs must preserve exception context for server-side debugging.
- `AppError` client errors below 500 should not be logged as actionable server errors.
- `SQLAlchemyError` and unhandled exceptions must be logged at error level with exception context.
- HTTP and request-validation exceptions may be logged at warning level.
- User-facing errors must not expose stack traces.
- Request summary logs must include enough information to investigate latency and failures.
- Request summary logs include HTTP metadata and SQL timing/count statistics collected during the request.
- Request summary logs are emitted once per request by `get_db()` when DB activity exists or by request middleware when no DB activity exists.
- Request summary logs include actor context when authentication sets it.
- Request summary logs should flag slow requests, slow queries, high query counts, and possible N+1 patterns.
- Slow-request, slow-query, and high-query-count thresholds are controlled by `LOG_SLOW_REQUEST_MS`, `LOG_SLOW_QUERY_MS`, and `LOG_HIGH_QUERY_COUNT`.
- Diagnostic routes such as `/`, `/health`, `/ready`, `/docs`, `/openapi.json`, and `/favicon.ico` may skip request summary logging.
- SQLAlchemy query start/end hooks record per-request SQL telemetry.
- SQL logging must be limited to development/debugging contexts.
- SQL logging is controlled by the `LOG_SQL` setting through the `sqlalchemy.engine` logger.
- SQL logs must not leak sensitive production data.
- Sentry or equivalent error reporting must receive unhandled server-side exceptions with request context when configured.
- Sentry is disabled in development by default and enabled only when configured.
- Product audit history and domain timelines are product behavior, not observability.
