## Scope

This file records a test-suite quality audit of:

- `backend/tests/`
- `backend/tax_rules_config/tests/`

It does not change product behavior or define new testing rules. Canonical testing rules remain in
`docs/workflow/testing.md`.

Source of truth: reference

# Backend Test Suite Audit

Audit date: 2026-06-05

## Method

- Read the canonical agent, architecture, testing, verification, security, and domain docs.
- Inspected all 349 Python files under both test roots.
- Compared suspicious tests with nearby tests, implementation paths, and documented domain rules.
- Ran pytest collection with the repo virtualenv.

Pytest originally collected **1,307 executable tests**. The larger function total in
`backend/defs-report.txt` includes helpers and is not the executable test count.

## Execution Status

Executed on 2026-06-05:

- Removed the safe-delete candidates: duplicate regression files, removed-route assertions,
  import/singleton/`repr` checks, dead dashboard coverage, private PDF helper tests, and tests that
  preserved the unimplemented reminder executor.
- Consolidated duplicate health, VAT due-date, advance-payment due-date/timing, business default,
  client validation, annual-report repository, and tax-calendar linked-row coverage.
- Rewrote root/info route coverage through the HTTP client.
- Rewrote notification trigger validation to construct the real service with the SQLite test
  session instead of bypassing initialization with `__new__` and six patched dependencies.
- Removed mock-heavy timeline orchestration duplicates while retaining DB-backed API ordering,
  pagination, and missing-client behavior.
- Replaced timeline API global mutation with `monkeypatch`, and made the tax-calendar fallback
  warning test deterministic.
- Moved the shared VAT item factory out of a test-discovery module into `tests/helpers/`.

Second execution pass on 2026-06-05:

- Deleted five signature-request auto-advance tests that mocked away annual-report readiness and
  therefore claimed behavior the real flow cannot currently complete.
- Deleted mock-only search service coverage, the exact-call-count recent-activity batching test,
  the exact-call-count work-queue batching test, and the thin permanent-document service
  list/delete duplicate.
- Deleted the remaining generic binder/charge regression file because owning API suites provide
  stronger coverage.
- Removed `repr`, unknown-field fallback, service-delegation, duplicate tax-calendar API
  calculation, duplicate task API happy-path, and duplicate dashboard item-shape assertions.
- Consolidated annual-report status orchestration from ten fragmented/mock-heavy tests into five
  behavioral tests, including a real SQLite-backed pending-client signature-request scenario.
- Consolidated VAT data-entry additional coverage from six tests to three domain-decision tests.
- Consolidated business repository legal-entity query semantics from fourteen fragmented tests to
  five scenarios.
- Consolidated action-builder coverage from ten micro-tests to three representative scenarios.

Final result: **1,196 tests collected and 1,196 passed**. The second pass removed **55 executable
tests**. Across both execution passes, the suite is down **111 executable tests** from the original
1,307 without removing security, permission, authentication, IDOR, ownership, cross-client
isolation, audit-integrity, or documented tax/domain-rule coverage.

Remaining justified targets:

- `tests/clients/service/test_client_service_mutations.py::test_list_clients_enriches_annual_turnover_with_batch_lookup`
  still asserts internal batching, but also remains the only focused check for reported-turnover
  precedence over manual turnover.
- `tests/annual_reports/api/test_annual_report_financials.py::test_auto_populate_response_contract_includes_skips_and_breakdown`
  uses a fake service and exact call capture, but remains the focused API response-contract and
  actor-forwarding check.
- Business create/default and lifecycle error tests still contain some repository replacement.
  They were retained where no equally strong API or real-DB edge-case coverage was found.
- Signature-request annual-report auto-advance has no trustworthy passing behavior test after the
  bug-preserving mock tests were deleted. Add real DB-backed coverage only after the readiness/order
  behavior is corrected.

## Findings Summary

The audit records 66 grouped findings:

| Classification | Count |
|---|---:|
| DUPLICATE | 5 |
| REDUNDANT | 11 |
| LOW_VALUE | 9 |
| OVER_SPECIFIED | 11 |
| OBSOLETE | 3 |
| WRONG_LAYER | 6 |
| FLAKY_RISK | 3 |
| KEEP | 18 |

Recommended actions:

| Safer action | Count |
|---|---:|
| delete | 16 |
| merge with another test | 6 |
| simplify | 9 |
| move to another layer | 2 |
| rewrite behaviorally | 15 |
| keep | 18 |

## Action Priority

1. Delete tests that only prove imports, `repr`, singleton existence, removed routes, or unrelated
   modules still load.
2. Merge duplicate due-date, health, regression, tax-calendar summary, and default-value coverage.
3. Rewrite mock-heavy service orchestration tests using the real SQLite session.
4. Remove tests that preserve documented broken or retired behavior.
5. Replace internal call-count assertions with behavioral or query-count assertions.

## Cross-Cutting, Regression, Core, And Actions

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| DUPLICATE | `tests/regression/test_vat_module_regressions.py` | Re-tests binder receive, charge creation, and health behavior already covered by their owning API suites. Loading the VAT module is not an isolated behavior under test. | delete |
| REDUNDANT | `tests/regression/test_core_regressions_binders_charges_notifications.py` | Repeats binder receive, open-binder list, charge create, and document-signal happy paths with weaker assertions than owning-domain tests. | merge with another test |
| KEEP | `tests/regression/test_readonly_endpoints_no_side_effects.py` | Protects a meaningful cross-domain regression: GET aggregation routes must not mutate DB state. | keep |
| LOW_VALUE | `tests/actions/test_action_registry.py::test_action_registry_exports_report_deadline_actions` | Only asserts an imported symbol is callable. | delete |
| REDUNDANT | `tests/core/test_action_builders.py` | Ten tiny tests separately assert direct argument/default copying. The behavior matters, but the fragmentation adds noise. | simplify |
| LOW_VALUE | `tests/core/test_config_additional.py::test_settings_singleton_exists` | Only asserts module import initialized an instance. | delete |
| WRONG_LAYER | `tests/core/test_main_additional.py::test_root_and_info_handlers_return_expected_payload` | Calls route handlers directly, bypassing routing and response behavior. | move to another layer |
| LOW_VALUE | `tests/users/models/test_user_model.py::test_user_repr_includes_key_fields` | Tests debug-string formatting with no domain or contract value. | delete |
| LOW_VALUE | `tests/businesses/test_constants.py` | Asserts a private constant imports values from another enum. | delete |
| REDUNDANT | removed-route tests in signature requests, notifications, and work queue | Tests only prove old routes return 404/405. They preserve absence rather than current contract behavior. | delete |

Removed-route tests:

- `tests/signature_requests/api/test_signature_requests.py::test_removed_send_endpoint_returns_404`
- `tests/signature_requests/api/test_signature_requests.py::test_removed_create_and_send_endpoint_is_not_available`
- `tests/notification/api/test_notifications.py::test_unread_count_route_gone`
- `tests/work_queue/test_work_queue_api.py::test_work_queue_summary_endpoint_removed`

## Health

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| REDUNDANT | `test_health_endpoint_returns_200`, `test_health_endpoint_is_unauthenticated`, `test_health_endpoint_verifies_database` | All three execute the same request; the first already proves the route is unauthenticated and DB-backed through its response. | merge with another test |
| KEEP | `/health` and `/ready` unhealthy-response tests | The endpoints are separate operational contracts and both must return 503 correctly. | keep |
| KEEP | repository query-error and service unexpected-exception tests | The domain docs explicitly document both defensive layers. | keep |

## VAT Reports

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| DUPLICATE | `test_vat_report_deadline_fields.py`, snapshot tests in `test_due_date_source_of_truth.py`, and `test_vat_report_queries.py` | Three files repeatedly assert that effective/original snapshot dates map to the same response fields. `test_vat_report_queries.py` is the strongest focused coverage; one DB-backed serializer test adds integration value. | merge with another test |
| OVER_SPECIFIED | `test_due_date_source_of_truth.py::test_serializer_routing_uses_snapshot_when_effective_set` | Calls the lower-level helper directly while claiming to test serializer routing. | rewrite behaviorally |
| WRONG_LAYER | `test_data_entry_service_additional.py::test_add_invoice_not_found_and_invalid_status` and `test_delete_invoice_not_found_paths` | Fake repositories test wiring that is already covered by real service/API paths. | rewrite behaviorally |
| OVER_SPECIFIED | `test_data_entry_service_additional.py::test_add_invoice_autofill_fields_for_income_and_expense` | Replaces repositories and recalculation internals, so it can pass while real persistence/orchestration breaks. | rewrite behaviorally |
| LOW_VALUE | `tests/vat_reports/service/test_vat_report_test_utils.py` | Contains only MagicMock factories but is named as a test module. | move to another layer |
| KEEP | `tests/vat_reports/api/test_vat_security_fixes.py` | Protects filing guards, error-code behavior, and cross-client business-activity ownership. | keep |
| KEEP | repository soft-delete/client-scope/query tests | These cover non-trivial query semantics and IDOR-sensitive scoping. | keep |

## Advance Payments

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| REDUNDANT | first two tests in `test_due_date_source_of_truth.py` | Both prove generated schedules copy a custom `TaxCalendarEntry.due_date`; one representative generated period is enough. Direct-create coverage is distinct. | merge with another test |
| REDUNDANT | overview-row timing tests in `test_inv05_timing_uses_effective_due_date.py` | Repeat the same timing matrix already asserted for `AdvancePaymentRow`. | simplify |
| OBSOLETE | three `due_date_effective=None` fallback tests in `test_inv05_timing_uses_effective_due_date.py` | Preserve fallback to the legacy `due_date`, while canonical docs say `due_date_effective` is the overdue source of truth and legacy `due_date` is planned for removal. | rewrite behaviorally |
| KEEP | override-reason, immutable-original-date, frequency, and effective-date tests | These encode documented invariants INV-05/INV-07 and model event guards. | keep |

## Annual Reports

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| OVER_SPECIFIED | `test_financial_line_repository_guards.py::test_unscoped_financial_line_repository_methods_do_not_exist` | Asserts class shape, not cross-report access behavior. The adjacent scoped-access test is the real security regression. | delete |
| OVER_SPECIFIED | `test_apply_updates_uses_preloaded_entity_without_reload` and `test_delete_line_uses_preloaded_entity_without_reload` | Patch a specific repository call to enforce an internal implementation choice. | rewrite behaviorally |
| WRONG_LAYER | mock-heavy transition tests in `test_status_service_additional.py` | Replace signature/readiness services and repositories despite the testing rule to use the real SQLite session for service orchestration. | rewrite behaviorally |
| OVER_SPECIFIED | mocked aggregation cases in `test_vat_import_service.py` | Audit/permission assertions are valuable, but aggregate mapping tests can pass while VAT query integration is broken. | simplify |
| LOW_VALUE | `_fmt`, `_r`, and `_get_font` tests in `test_annual_report_pdf_service.py` | Assert private formatting/font helpers rather than the exported artifact. | delete |
| FLAKY_RISK | conditional PDF test that skips when `reportlab` is unavailable | Environment-dependent pass/skip behavior weakens CI confidence. The dependency should be fixed for the test environment. | simplify |
| KEEP | financial audit snapshots, mutation guards, VAT client-wide aggregation, status lifecycle, and tax engine tests | Encode documented financial/security/domain rules. | keep |

## Businesses And Clients

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| DUPLICATE | the two `create_business` default-`opened_at` tests | Same behavior appears in `test_business_service.py` and `test_business_service_additional.py`; the patched-date version is deterministic and stronger. | delete |
| DUPLICATE | the two `test_create_client_rejects_employee_entity_type` tests | The `CreateClientService` path is closer to real behavior and subsumes the identity-helper test. | delete |
| OVER_SPECIFIED | business-service tests that replace repositories with `SimpleNamespace` | Violate the project service-test convention and can miss real query/flush behavior. | rewrite behaviorally |
| OVER_SPECIFIED | client/correspondence repository `update_ignores_unknown_fields` tests | Preserve silent dropping of invalid internal kwargs; public schemas should reject unknown/invalid input instead. | rewrite behaviorally |
| REDUNDANT | fourteen tiny tests in `test_business_repository_legal_entity.py` | Query semantics are worth keeping, but true/false/empty cases are excessively fragmented. | merge with another test |
| KEEP | client identity graph, lifecycle cascades, savepoint behavior, and cross-client guards | High-value documented orchestration and ownership coverage. | keep |

## Tax Calendar

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| WRONG_LAYER | API and service copies of summary count/fallback-warning tests | API tests repeat service calculations instead of limiting themselves to route contract/auth/validation. | simplify |
| FLAKY_RISK | tests that branch or skip based on `registry_periodic_calendar_available(2027)` | Tests silently change meaning when config gains 2027 data, hiding regressions. | rewrite behaviorally |
| DUPLICATE | `test_annual_report_linked_item_appears` in grouped-items and grouped-links API files | Both prove the same annual linked-row aggregation with overlapping assertions. | merge with another test |
| KEEP | model constraints, rule overlap/versioning, materialization, link conflict, effective-date aggregation, and role tests | These encode central documented tax-calendar invariants. | keep |

## Reminders

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| OBSOLETE | `test_fire_due_ignores_future_and_terminal_reminders`, `test_fire_due_is_idempotent_for_failed_reminders`, `test_send_notification_failure_reason_is_not_delivery_failure` | Assert the known broken executor stub always fails with “not implemented.” Canonical docs call this a bug, not intended behavior. | delete |
| KEEP | create/list/enrichment/cancel/query tests | They cover the currently supported persistence and lifecycle contract. | keep |

## Dashboard, Timeline, Work Queue, Search, And Reports

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| OBSOLETE | `dashboard/service/test_advisor_today_service.py` | Only asserts a removed/dead behavior returns an empty list. | delete |
| LOW_VALUE | `dashboard_attention_service.py::test_returns_list_for_advisor` | Merely checks the result is a list; neighboring tests prove meaningful contents and role behavior. | delete |
| OVER_SPECIFIED | `timeline/service/test_timeline_service_get_client_timeline.py` | Heavy repository/private-method patching duplicates DB-backed API behavior and violates service-test conventions. | rewrite behaviorally |
| OVER_SPECIFIED | `test_append_lifecycle_change_events_skips_noise_and_keeps_meaningful` | Calls a private assembly helper directly. | rewrite behaviorally |
| FLAKY_RISK | timeline API tests manually replace and restore a module function | Manual global mutation is less reliable than the `monkeypatch` fixture when assertions/setup fail. | simplify |
| OVER_SPECIFIED | batch-loading call-count tests in dashboard, work queue, clients, and document search | Assert exact helper invocation counts/internal methods rather than bounded SQL/query behavior. | rewrite behaviorally |
| LOW_VALUE | `search/api/test_search.py::test_search_with_query` | Only asserts 200 and adds nothing beyond the endpoint smoke test. | delete |
| WRONG_LAYER | mock-only search service tests | Replace all repositories, testing DTO wiring instead of real filtering/aggregation. | rewrite behaviorally |
| REDUNDANT | aging bucket totals asserted at both service and API layers | Keep detailed bucket decisions at service layer; API should assert contract/serialization/auth with a smaller scenario. | simplify |
| KEEP | aging cap, exports, work-queue merge/filter/order semantics, read-only behavior, and source scoping | Non-trivial aggregation and documented cross-domain behavior. | keep |

Batch-loading tests to rewrite:

- `tests/dashboard/service/test_recent_activity_service.py::test_recent_activity_batches_related_entity_lookups`
- `tests/work_queue/test_work_queue_service.py::test_work_queue_batch_loads_charge_client_identities_once`
- `tests/clients/service/test_client_service_mutations.py::test_list_clients_enriches_annual_turnover_with_batch_lookup`
- `tests/search/service/test_document_search_service.py::test_document_search_builds_results_and_caches_client_lookup`

## Notifications, Signature Requests, And Permanent Documents

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| OVER_SPECIFIED | notification trigger-validation tests built through `NotificationSendService.__new__` with six patched dependencies | Bypasses service initialization and can survive broken constructor wiring. | rewrite behaviorally |
| REDUNDANT | separate preview/send tests for each missing-entity trigger | Same validation matrix is repeated verbatim. | simplify |
| WRONG_LAYER | `signature_requests/service/test_signer_actions_auto_advance.py` | Almost entirely fake orchestration/private-method testing; real sign + annual-report transition behavior should use DB-backed integration. | rewrite behaviorally |
| REDUNDANT | permanent-document service list/version tests plus API list/version test | Service methods are thin delegation; API integration covers the meaningful contract. | delete |
| KEEP | notification policy/cooldown/idempotency, signature lifecycle/token, permanent-document ownership and authorization tests | Security and documented lifecycle behavior. | keep |

## Invoice, Audit, Notes, Tasks, Users, And Tax Rules

| Classification | Test(s) | Why suspicious | Safer action |
|---|---|---|---|
| LOW_VALUE | invoice repository `repr(created)` assertion | Debug representation is unrelated to repository query semantics. | delete |
| REDUNDANT | task API happy paths that repeat detailed service lifecycle decisions | Keep API contract/status/auth assertions concise; keep lifecycle decision matrix in service tests. | simplify |
| KEEP | task terminal-state/source-link/work-queue behavior | Encodes documented domain decisions and cross-domain linking rules. | keep |
| KEEP | audit serialization/append-only/read contract tests | Audit payload fidelity and actor handling are accountability behavior. | keep |
| KEEP | auth token invalidation, cookie-vs-bearer, role, permission, IDOR, and cross-client tests | Security-sensitive tests must remain unless demonstrably duplicated. | keep |
| KEEP | `tax_rules_config/tests/test_tax_rules_config.py` | Short constant/rate/deadline tests encode external tax policy and supported-year rules, not incidental constants. | keep |

## Documentation/Test Conflicts

These are not deletion candidates. They indicate canonical docs should be re-audited:

| Classification | Test(s) | Conflict | Safer action |
|---|---|---|---|
| KEEP | `tests/work_queue/test_work_queue_service.py::test_work_queue_advance_payment_uses_effective_due_date_when_set` | Test and current code indicate the known work-queue legacy-date bug is fixed, while `docs/domains/work-queue.md` still lists it as current. | keep |
| KEEP | `tests/notes/test_entity_note_service.py::test_list_client_notes_requires_existing_client` and `test_add_client_note_requires_existing_client` | Tests and current code indicate client existence guards are implemented, while `docs/domains/notes.md` still lists their absence as current. | keep |

## Recommended Cleanup Batches

### Batch 1: Safe Deletes

- Removed-route tests.
- Import/callable/singleton/`repr` tests.
- Generic VAT/core regression duplicates.
- Empty advisor-today test.
- Private PDF helper tests.
- Reminder tests that preserve the unimplemented executor.

### Batch 2: Merge And Simplify

- VAT deadline snapshot tests.
- Advance-payment timing/source-of-truth tests.
- Health happy-path tests.
- Business repository scenario tests.
- Tax-calendar summary API/service duplicates.
- Action-builder tiny tests.
- Task API happy-path duplication.

### Batch 3: Behavioral Rewrites

- Timeline orchestration.
- Signature-request auto-advance.
- Annual-report status orchestration.
- VAT data-entry fake-repository tests.
- Notification `__new__` trigger validation.
- Search mock-only service tests.
- Internal batch-call-count tests.

## Verification

Executed:

```text
JWT_SECRET=test-secret-minimum-32-bytes-value ./.venv/bin/python -m pytest --collect-only -q
JWT_SECRET=test-secret-minimum-32-bytes-value ./.venv/bin/python -m pytest -q
```

Result:

- **1,196 tests collected in 0.63s**
- **1,196 passed in 520.80s**
- `git diff --check`: passed
