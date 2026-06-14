## Scope

This file owns only:

- The per-endpoint error-response documentation matrix for OpenAPI (#53).
- The authoritative list of which application error statuses each endpoint documents, with raise-site evidence.

This file must not contain:

- Error contract rules (those live in `docs/architecture/api-contracts.md`).
- Runtime exception-handling behavior.

Source of truth: reference (subordinate to `docs/architecture/api-contracts.md`)

# Error-Doc Matrix (#53)

Authoritative mapping of documented application error statuses per endpoint. Generated from real raise sites in `backend/app`, verified against `app/core/exception_handlers.py`. The coverage matrix test (`backend/tests/core/test_error_doc_coverage_matrix.py`) asserts the spec matches this matrix.

## Mechanism split

Statuses are documented by two mechanisms — do not duplicate:

- **`401` and `403` — mostly injected globally in `app/core/openapi.py` (`build_openapi`)**:
  - `401`: every non-public operation (public set = `app/core/public_endpoints.py`, 11 ops).
  - `403`: every operation whose route has a `require_role(...)` dependency (detected by walking `route.dependant` and checking the dependency callable's `__requires_role__` marker). **186 of 201 ops are role-gated.**
  - The non-public ops WITHOUT detected `require_role` are `GET /api/v1/auth/me`, `POST /api/v1/auth/logout`, `GET /api/v1/reminders/{reminder_id}`, and `POST /api/v1/reminders/{reminder_id}/cancel`.
  - `GET /api/v1/reminders/{reminder_id}` and `POST /api/v1/reminders/{reminder_id}/cancel` document per-route `forbidden_response()` because they are authenticated but not role-gated.
- **`400`, `404`, `409`, `500` — documented per-route** via the `responses=` helpers in `app/core/openapi_responses.py`:
  - `404`: already swept (ID-param rule, guarded by `tests/core/test_openapi_not_found_docs.py`). Not re-listed here unless new.
  - `400` / `409` / `500`: added by the #53 sweep per the table below.

## Status → handler mapping (verified, `exception_handlers.py`)

| status | source |
|---|---|
| 400 | base `AppError(status_code=400)`; runtime service/router `ValueError` (`value_error_handler`, :120) |
| 403 | `ForbiddenError`; `require_role` HTTPException |
| 404 | `NotFoundError` |
| 409 | `ConflictError` |
| 500 | explicit `AppError(status_code=500)` only — documented for `DOCUMENT.UPLOAD_FAILED` |
| 422 | `RequestValidationError` → `HTTPValidationError` (out of #53 scope) |

## 500 policy

Document `500` **only** at `permanent_document_service.py:178` (`AppError("DOCUMENT.UPLOAD_FAILED", status_code=500)`), reachable from `POST /documents/upload` and `PUT /documents/client/{id}/{document_id}/replace`.

**Intentionally NOT documented as 500** (environment/optional-dependency `ImportError` guards, not a business contract): `GET /clients/export`, `GET /clients/template` (`clients_excel.py:40,59`), `GET /reports/aging/export` (`reports_export_service.py:39,44`). The generic `SQLAlchemyError`/`Exception` → 500 handler is global and is not documented per-endpoint.

## Per-route sweep matrix (400 / 403 / 409 / 500)

Legend: ✓ = document this status on the endpoint. Blank = not applicable. `H` = handled by global hook (401 all non-public; 403 all role-gated) — not listed per row. Per-route 403 is listed only for authenticated, non-role-gated endpoints that can still forbid access.

### charges (reference impl — already complete)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /charges | ✓ | | |
| POST /charges/{id}/cancel | ✓ | ✓ | |
| POST /charges/{id}/issue | ✓ | ✓ | |
| POST /charges/{id}/mark-paid | ✓ | ✓ | |

### vat (conflict: intake.py, data_entry_invoice*.py; 400: data_entry_*, filing, intake, period_options)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /vat/work-items | ✓ | ✓ | |
| PATCH /vat/work-items/{id} | ✓ | | |
| DELETE /vat/work-items/{id} | ✓ | | |
| POST /vat/work-items/{id}/invoices | ✓ | ✓ | |
| PATCH /vat/work-items/{id}/invoices/{invoice_id} | ✓ | ✓ | |
| DELETE /vat/work-items/{id}/invoices/{invoice_id} | ✓ | | |
| POST /vat/work-items/{id}/file | ✓ | ✓ | |
| POST /vat/work-items/{id}/materials-complete | ✓ | ✓ | |
| POST /vat/work-items/{id}/ready-for-review | ✓ | ✓ | |
| POST /vat/work-items/{id}/send-back | ✓ | ✓ | |

### binders (400: handover/intake/lifecycle services)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /binders/receive | ✓ | | |
| POST /binders/{id}/handover-to-client | ✓ | | |
| POST /binders/{id}/mark-full | ✓ | | |
| POST /binders/{id}/mark-ready-for-handover | ✓ | | |
| POST /binders/{id}/receive-material | ✓ | | |
| POST /binders/{id}/reopen-capacity | ✓ | | |
| POST /binders/{id}/revert-ready-for-handover | ✓ | | |
| PATCH /binders/{id}/intakes/{intake_id} | ✓ | | |
| POST /binders/handover-to-client-bulk | ✓ | | |
| POST /binders/mark-ready-for-handover-bulk | ✓ | | |

### annual_reports (409: create/status/vat_import; 400: many services)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /annual-reports | ✓ | ✓ | |
| POST /annual-reports/{id}/status | ✓ | ✓ | |
| POST /annual-reports/{id}/transition | ✓ | ✓ | |
| POST /annual-reports/{id}/submit | ✓ | ✓ | |
| POST /annual-reports/{id}/amend | ✓ | ✓ | |
| POST /annual-reports/{id}/auto-populate | ✓ | | |
| POST /annual-reports/{id}/deadline | ✓ | | |
| POST /annual-reports/{id}/expenses | ✓ | | |
| PATCH /annual-reports/{id}/expenses/{line_id} | ✓ | | |
| POST /annual-reports/{id}/income | ✓ | | |
| PATCH /annual-reports/{id}/income/{line_id} | ✓ | | |
| POST /annual-reports/{id}/annex/{schedule} | ✓ | | |
| PATCH /annual-reports/{id}/annex/{schedule}/{line_id} | ✓ | | |
| POST /annual-reports/{id}/schedules | ✓ | | |
| POST /annual-reports/{id}/schedules/complete | ✓ | | |
| POST /annual-reports/{id}/tax-calculation/save | ✓ | | |
| PATCH /annual-reports/{id}/details | ✓ | | |

### clients (409: create/lifecycle/guards; 400: create)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients | ✓ | ✓ | |
| PATCH /clients/{id} | ✓ | ✓ | |
| DELETE /clients/{id} | | ✓ | |
| POST /clients/{id}/restore | | ✓ | |
| POST /clients/import | ✓ | | |

### businesses (409: business_service/lifecycle; 400: business_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients/{id}/businesses | ✓ | ✓ | |
| PATCH /clients/{id}/businesses/{business_id} | ✓ | ✓ | |
| DELETE /clients/{id}/businesses/{business_id} | | ✓ | |
| POST /clients/{id}/businesses/{business_id}/restore | | ✓ | |

### tasks (409 + 400: task_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /tasks | ✓ | | |
| POST /tasks/bulk-complete | ✓ | ✓ | |
| POST /tasks/bulk-assign | ✓ | ✓ | |
| PATCH /tasks/{id} | ✓ | ✓ | |
| POST /tasks/{id}/complete | | ✓ | |
| POST /tasks/{id}/cancel | | ✓ | |
| GET /clients/{id}/tasks | | | |

### invoices (409 + 400: invoice_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /invoices | ✓ | ✓ | |

### advance_payments (409 + 400: advance_payment_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients/{id}/advance-payments | ✓ | ✓ | |
| POST /clients/{id}/advance-payments/generate | ✓ | ✓ | |
| PATCH /clients/{id}/advance-payments/{payment_id} | ✓ | ✓ | |

### documents/permanent_documents (400 + 500 upload)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /documents/upload | ✓ | | ✓ |
| PUT /documents/client/{id}/{document_id}/replace | ✓ | | ✓ |
| DELETE /documents/client/{id}/{document_id} | ✓ | | |

### tax_calendar (409 + 400: materialization_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /settings/tax-calendar/bootstrap | ✓ | ✓ | |

### users (409 + 400: user_management_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /users | ✓ | ✓ | |
| PATCH /users/{id} | ✓ | ✓ | |
| POST /users/{id}/activate | | ✓ | |
| POST /users/{id}/deactivate | | ✓ | |
| POST /users/{id}/reset-password | ✓ | | |

### communications (400: correspondence schema; 403 via service handled by hook+role)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients/{id}/correspondence | ✓ | | |
| PATCH /clients/{id}/correspondence/{correspondence_id} | ✓ | | |

### notes (400: entity_note_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients/{id}/notes | ✓ | | |
| PATCH /clients/{id}/notes/{note_id} | ✓ | | |
| POST /clients/{id}/businesses/{business_id}/notes | ✓ | | |
| PATCH /clients/{id}/businesses/{business_id}/notes/{note_id} | ✓ | | |

### authority_contacts (400: schema)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /clients/{id}/authority-contacts | ✓ | | |
| PATCH /clients/{id}/authority-contacts/{contact_id} | ✓ | | |

### notifications (400: notification services)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /notifications/preview | ✓ | | |
| POST /notifications/send | ✓ | | |

### reminders (400: reminder_service)
| method path | 400 | 403 | 409 | 500 |
|---|---|---|---|---|
| POST /reminders/ | ✓ | | | |
| GET /reminders/{id} | | ✓ | | |
| POST /reminders/{id}/cancel | | ✓ | ✓ | |

### audit (400: audit_trail_service)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| GET /audit/{entity_type}/{entity_id} | ✓ | | |

### signature_requests (400: signature services)
| method path | 400 | 409 | 500 |
|---|---|---|---|
| POST /signature-requests | ✓ | | |
| POST /clients/{id}/signature-requests/{request_id}/cancel | | ✓ | |

### auth / password_reset (400: auth_service, password_reset_service; 401 on credential failure)
| method path | 400 | 401 (explicit) | 500 |
|---|---|---|---|
| POST /auth/login | ✓ | ✓ | |
| POST /auth/refresh | | ✓ | |
| POST /auth/reset-password | ✓ | | |

Note: `auth` endpoints are public (no `require_role`), so the global 401/403 hook does NOT cover them; `login`/`refresh` document `401` explicitly here for invalid-credential / invalid-token.

## Intentional exclusions (allowlist for coverage test)

- **No 500** on `GET /clients/export`, `GET /clients/template`, `GET /reports/aging/export` — `ImportError` env-guards, not a business contract.
- **No 403 per-route** except `GET /reminders/{id}` and `POST /reminders/{id}/cancel` — all role-gating is hook-injected; `GET /auth/me`, `POST /auth/logout` are role-free and have no business `ForbiddenError`.
- Read-only list/summary/dashboard endpoints document no `400/409` (no reachable business error beyond auth + not-found).
