## Scope
This file owns only:
- A single index of backend environment/config variables and where they are defined.

This file must not contain:
- Secret values.
- Architecture, security, or workflow rules (link to the owning docs instead).

Source of truth: reference

# Config Reference

This file is non-normative. The authoritative config is the `Settings` class in `backend/app/config.py`; defaults, validation, and per-env overrides live there. This page is an index so the variables are not scattered across workflow docs.

## Env loading

- `APP_ENV` selects the environment: `development` (alias `local`), `test`, `staging`, `production`. Default `development`.
- `ENV_FILE` loads a specific dotenv file; otherwise `backend/.env.{APP_ENV}` is loaded if present.
- Existing process env vars win over dotenv files (`override=False`).

## Core

| Var | Default | Purpose |
|-----|---------|---------|
| `APP_ENV` | `development` | Active environment. |
| `PORT` | `8000` | HTTP port. |
| `DATABASE_URL` | postgres local (sqlite in test) | DB connection string. Must not be localhost in staging/production. |

## Auth / security

See `docs/architecture/security.md` for the rules behind these.

| Var | Default | Purpose |
|-----|---------|---------|
| `JWT_SECRET` | — (required) | JWT signing secret. App fails to start if unset. |
| `JWT_ALGORITHM` | `HS256` | JWT algorithm. |
| `AUTH_ACCESS_TOKEN_EXPIRE_MINUTES` | `15` | Access-token lifetime. |
| `AUTH_REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Refresh-token lifetime. |
| `REFRESH_COOKIE_NAME` | `refresh_token` | Refresh cookie name. |
| `REFRESH_COOKIE_PATH` | `/api/v1/auth` | Refresh cookie path. |
| `REFRESH_COOKIE_SAMESITE` | `lax` (`none` in production) | Refresh cookie SameSite. |
| `PASSWORD_RESET_TOKEN_EXPIRE_MINUTES` | `5` | Password-reset token lifetime. |
| `PASSWORD_RESET_DEV_LOG` | `false` | Log reset tokens (dev only). |
| `AUTH_LOGIN_RATE_LIMIT` | `5/minute` (`10000/minute` in test) | Login rate limit. |

## CORS / frontend

| Var | Default | Purpose |
|-----|---------|---------|
| `CORS_ALLOWED_ORIGINS` | localhost dev origins | Comma-separated allowed origins. Required in staging/production. |
| `FRONTEND_BASE_URL` | `http://localhost:5173` | Frontend base URL. |
| `FRONTEND_PASSWORD_RESET_URL` | `http://localhost:5173/reset-password` | Reset-link target. |

## Logging / observability

See `docs/architecture/observability.md`.

| Var | Default | Purpose |
|-----|---------|---------|
| `LOG_LEVEL` | `INFO` | Log level. |
| `LOG_FORMAT` | `text` (`json` in staging/production) | Log output format. Must be `json` in staging/production. |
| `LOG_SQL` | `false` (`true` in development) | Echo SQL. |
| `LOG_SLOW_REQUEST_MS` | `500` | Slow-request threshold. |
| `LOG_SLOW_QUERY_MS` | `250` | Slow-query threshold. |
| `LOG_HIGH_QUERY_COUNT` | `20` | High query-count warning threshold. |
| `SENTRY_ENABLED` | `false` | Enable Sentry. |
| `SENTRY_DSN` | — | Sentry DSN. Required when `SENTRY_ENABLED` in staging/production. |
| `SENTRY_ENVIRONMENT` | `APP_ENV` | Sentry environment tag. |
| `SENTRY_TRACES_SAMPLE_RATE` | `0.0` | Sentry trace sampling. |

## Notifications / external providers

| Var | Default | Purpose |
|-----|---------|---------|
| `NOTIFICATIONS_ENABLED` | `false` | Master switch for outbound notifications. |
| `BREVO_API_KEY` | — | Email provider (Brevo) key. |
| `BREVO_API_URL` | Brevo SMTP endpoint | Email provider URL. |
| `EMAIL_FROM_ADDRESS` | — | Sender address. |
| `EMAIL_FROM_NAME` | — | Sender name. |
| `WHATSAPP_API_KEY` | — | WhatsApp (360dialog) key. |
| `WHATSAPP_API_URL` | 360dialog endpoint | WhatsApp API URL. |
| `WHATSAPP_FROM_NUMBER` | — | WhatsApp sender number. |
| `INVOICE_PROVIDER_BASE_URL` | — | Invoice provider base URL. |
| `INVOICE_PROVIDER_API_KEY` | — | Invoice provider key. |

## Storage

| Var | Default | Purpose |
|-----|---------|---------|
| `R2_ACCESS_KEY_ID` | — | Object storage (Cloudflare R2) access key. |
| `R2_SECRET_ACCESS_KEY` | — | R2 secret key. |
| `R2_BUCKET_NAME` | — | R2 bucket. |
| `R2_ENDPOINT_URL` | — | R2 endpoint. |
| `R2_REGION` | `auto` | R2 region. |
| `LOCAL_STORAGE_PATH` | `./storage` | Local file storage path (non-R2). |

## Notes

- Secrets (`JWT_SECRET`, provider keys, `R2_*`) come from the environment / dotenv files, never committed.
- When adding a setting to `backend/app/config.py`, add it to the matching table here in the same change.
