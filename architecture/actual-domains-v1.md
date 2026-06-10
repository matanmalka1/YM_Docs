# Actual CRM Domains (v1)

> Generated from a scan of `backend/app/` as it exists today.
> Reflects only what is present in code — endpoints, public service functions, and public repository functions.
> Skipped infrastructure folders: `common/`, `core/`, `middleware/`, `infrastructure/`, `seed/`, `utils/`, `health/`.
> Private functions (`_`-prefixed) and `__init__` are excluded.

---

# actions

## API Endpoints
No API layer

## Service Functions
| Function | File |
|---|---|
| build_confirm | [action_helpers.py](../../backend/app/actions/services/action_helpers.py) |
| build_action | [action_helpers.py](../../backend/app/actions/services/action_helpers.py) |
| get_binder_actions_for_state | [binder_actions.py](../../backend/app/actions/services/binder_actions.py) |
| get_binder_actions | [binder_actions.py](../../backend/app/actions/services/binder_actions.py) |
| get_business_actions | [business_actions.py](../../backend/app/actions/services/business_actions.py) |
| get_charge_actions | [charge_actions.py](../../backend/app/actions/services/charge_actions.py) |
| total_created | [obligation_orchestrator.py](../../backend/app/actions/services/obligation_orchestrator.py) |
| generate_client_obligations | [obligation_orchestrator.py](../../backend/app/actions/services/obligation_orchestrator.py) |
| generate_client_obligations_result | [obligation_orchestrator.py](../../backend/app/actions/services/obligation_orchestrator.py) |
| obligation_fields_changed | [obligation_orchestrator.py](../../backend/app/actions/services/obligation_orchestrator.py) |
| get_annual_report_actions | [report_deadline_actions.py](../../backend/app/actions/services/report_deadline_actions.py) |
| get_vat_work_item_actions | [vat_report_actions.py](../../backend/app/actions/services/vat_report_actions.py) |

## Repository Functions
No repository layer

---

# advance_payments

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | /generate | generate_advance_payment_schedule | [advance_payment_generate.py](../../backend/app/advance_payments/api/advance_payment_generate.py) |
| GET | /overview | list_advance_payments_overview | [advance_payments_overview.py](../../backend/app/advance_payments/api/advance_payments_overview.py) |
| GET | /overview/batches | list_advance_payment_batches | [advance_payments_overview.py](../../backend/app/advance_payments/api/advance_payments_overview.py) |
| GET | (root) | list_advance_payments | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |
| POST | (root) | create_advance_payment | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |
| GET | /prefill-turnover | get_prefill_turnover | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |
| GET | /kpi | get_annual_kpis | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |
| PATCH | /{payment_id} | update_advance_payment | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |
| DELETE | /{payment_id} | delete_advance_payment | [advance_payments.py](../../backend/app/advance_payments/api/advance_payments.py) |

## Service Functions
| Function | File |
|---|---|
| list_overview | [advance_payment_analytics_service.py](../../backend/app/advance_payments/services/advance_payment_analytics_service.py) |
| get_annual_kpis_for_client | [advance_payment_analytics_service.py](../../backend/app/advance_payments/services/advance_payment_analytics_service.py) |
| get_overview_kpis | [advance_payment_analytics_service.py](../../backend/app/advance_payments/services/advance_payment_analytics_service.py) |
| get_month_batches | [advance_payment_analytics_service.py](../../backend/app/advance_payments/services/advance_payment_analytics_service.py) |
| derive_annual_income_from_vat | [advance_payment_calculator.py](../../backend/app/advance_payments/services/advance_payment_calculator.py) |
| calculate_expected_amount | [advance_payment_calculator.py](../../backend/app/advance_payments/services/advance_payment_calculator.py) |
| generate_annual_schedule | [advance_payment_generator.py](../../backend/app/advance_payments/services/advance_payment_generator.py) |
| default_period_months_count_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| list_payments_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| create_payment_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| update_payment_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| delete_payment_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| generate_annual_schedule | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| get_prefill_turnover_for_client | [advance_payment_service.py](../../backend/app/advance_payments/services/advance_payment_service.py) |
| get_period_start_months | [constants.py](../../backend/app/advance_payments/services/constants.py) |

## Repository Functions
| Function | File |
|---|---|
| advance_payment_start_month_expr | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| advance_payment_year_range_filter | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| advance_payment_matches_month_expr | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| list_overview_payments | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| list_overview_payment_rows | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| sum_paid_by_client_year | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| get_collections_aggregates | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| get_annual_kpis_for_client | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| get_overview_kpis | [advance_payment_aggregation_repository.py](../../backend/app/advance_payments/repositories/advance_payment_aggregation_repository.py) |
| batch_summary_by_month | [advance_payment_batch_repository.py](../../backend/app/advance_payments/repositories/advance_payment_batch_repository.py) |
| completion_for_period | [advance_payment_dashboard_repository.py](../../backend/app/advance_payments/repositories/advance_payment_dashboard_repository.py) |
| completion_for_periods | [advance_payment_dashboard_repository.py](../../backend/app/advance_payments/repositories/advance_payment_dashboard_repository.py) |
| create | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| get_by_id_for_client_record | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| list_by_client_record_year | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| exists_for_period | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| get_by_period | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| update_payment | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| soft_delete | [advance_payment_repository.py](../../backend/app/advance_payments/repositories/advance_payment_repository.py) |
| get_turnover_for_period | [turnover_lookup_repository.py](../../backend/app/advance_payments/repositories/turnover_lookup_repository.py) |
| get_turnover_for_many | [turnover_lookup_repository.py](../../backend/app/advance_payments/repositories/turnover_lookup_repository.py) |
| get_turnover_for_many_clients | [turnover_lookup_repository.py](../../backend/app/advance_payments/repositories/turnover_lookup_repository.py) |
| get_prefill_turnover | [turnover_lookup_repository.py](../../backend/app/advance_payments/repositories/turnover_lookup_repository.py) |

---

# alerts

## API Endpoints
No API layer

## Service Functions
| Function | File |
|---|---|
| build | [alert_service.py](../../backend/app/alerts/services/alert_service.py) |

## Repository Functions
No repository layer

---

# annual_reports

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{report_id}/annex/{schedule} | list_annex_lines | [annual_report_annex.py](../../backend/app/annual_reports/api/annual_report_annex.py) |
| POST | /{report_id}/annex/{schedule} | add_annex_line | [annual_report_annex.py](../../backend/app/annual_reports/api/annual_report_annex.py) |
| PATCH | /{report_id}/annex/{schedule}/{line_id} | update_annex_line | [annual_report_annex.py](../../backend/app/annual_reports/api/annual_report_annex.py) |
| DELETE | /{report_id}/annex/{schedule}/{line_id} | delete_annex_line | [annual_report_annex.py](../../backend/app/annual_reports/api/annual_report_annex.py) |
| GET | /{report_id}/charges | list_report_charges | [annual_report_charges.py](../../backend/app/annual_reports/api/annual_report_charges.py) |
| POST | (root) | create_annual_report | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| GET | (root) | list_annual_reports | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| GET | /overdue | list_overdue | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| GET | /{report_id} | get_annual_report | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| DELETE | /{report_id} | delete_annual_report | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| POST | /{report_id}/amend | amend_annual_report | [annual_report_create_read.py](../../backend/app/annual_reports/api/annual_report_create_read.py) |
| GET | /{report_id}/details | get_annual_report_detail | [annual_report_detail.py](../../backend/app/annual_reports/api/annual_report_detail.py) |
| PATCH | /{report_id}/details | update_annual_report_detail | [annual_report_detail.py](../../backend/app/annual_reports/api/annual_report_detail.py) |
| POST | /tax-preview | get_tax_preview | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| GET | /{report_id}/financials | get_financial_summary | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| GET | /{report_id}/tax-calculation | get_tax_calculation | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| GET | /{report_id}/advances-summary | get_advances_summary | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| GET | /{report_id}/readiness | get_readiness_check | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| POST | /{report_id}/income | add_income_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| PATCH | /{report_id}/income/{line_id} | update_income_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| DELETE | /{report_id}/income/{line_id} | delete_income_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| POST | /{report_id}/expenses | add_expense_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| PATCH | /{report_id}/expenses/{line_id} | update_expense_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| DELETE | /{report_id}/expenses/{line_id} | delete_expense_line | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| POST | /{report_id}/auto-populate | auto_populate_from_vat | [annual_report_financials.py](../../backend/app/annual_reports/api/annual_report_financials.py) |
| POST | /{report_id}/schedules | add_schedule | [annual_report_schedule.py](../../backend/app/annual_reports/api/annual_report_schedule.py) |
| GET | /{report_id}/schedules | list_schedules | [annual_report_schedule.py](../../backend/app/annual_reports/api/annual_report_schedule.py) |
| POST | /{report_id}/schedules/complete | complete_schedule | [annual_report_schedule.py](../../backend/app/annual_reports/api/annual_report_schedule.py) |
| POST | /{report_id}/transition | transition_stage | [annual_report_stage_transition.py](../../backend/app/annual_reports/api/annual_report_stage_transition.py) |
| POST | /{report_id}/status | transition_status | [annual_report_status.py](../../backend/app/annual_reports/api/annual_report_status.py) |
| POST | /{report_id}/submit | submit_report | [annual_report_status.py](../../backend/app/annual_reports/api/annual_report_status.py) |
| POST | /{report_id}/deadline | update_deadline | [annual_report_status.py](../../backend/app/annual_reports/api/annual_report_status.py) |
| GET | /{report_id}/history | get_status_history | [annual_report_status.py](../../backend/app/annual_reports/api/annual_report_status.py) |
| POST | /{report_id}/tax-calculation/save | save_tax_calculation | [annual_report_tax.py](../../backend/app/annual_reports/api/annual_report_tax.py) |
| GET | /{report_id}/export/pdf | export_annual_report_pdf | [routes_export.py](../../backend/app/annual_reports/api/routes_export.py) |
| GET | /{client_record_id}/annual-reports | list_client_reports | [annual_report_client.py](../../backend/app/annual_reports/api/annual_report_client.py) |
| GET | /active/reports | list_active_season_reports | [annual_report_season.py](../../backend/app/annual_reports/api/annual_report_season.py) |
| GET | /active/summary | get_active_season_summary | [annual_report_season.py](../../backend/app/annual_reports/api/annual_report_season.py) |
| GET | /default | get_default_tax_year | [annual_report_season.py](../../backend/app/annual_reports/api/annual_report_season.py) |
| GET | /{tax_year}/reports | list_season_reports | [annual_report_season.py](../../backend/app/annual_reports/api/annual_report_season.py) |
| GET | /{tax_year}/summary | get_season_summary | [annual_report_season.py](../../backend/app/annual_reports/api/annual_report_season.py) |

## Service Functions
| Function | File |
|---|---|
| get_advances_summary | [advances_summary_service.py](../../backend/app/annual_reports/services/advances_summary_service.py) |
| get_advances_summary_for_report | [advances_summary_service.py](../../backend/app/annual_reports/services/advances_summary_service.py) |
| get_annex_lines | [annex_service.py](../../backend/app/annual_reports/services/annex_service.py) |
| add_annex_line | [annex_service.py](../../backend/app/annual_reports/services/annex_service.py) |
| update_annex_line | [annex_service.py](../../backend/app/annual_reports/services/annex_service.py) |
| delete_annex_line | [annex_service.py](../../backend/app/annual_reports/services/annex_service.py) |
| list_charges | [annual_report_charge_service.py](../../backend/app/annual_reports/services/annual_report_charge_service.py) |
| build_pdf | [annual_report_pdf_builder.py](../../backend/app/annual_reports/services/annual_report_pdf_builder.py) |
| p | [annual_report_pdf_builder.py](../../backend/app/annual_reports/services/annual_report_pdf_builder.py) |
| amount | [annual_report_pdf_builder.py](../../backend/app/annual_reports/services/annual_report_pdf_builder.py) |
| make_table | [annual_report_pdf_builder.py](../../backend/app/annual_reports/services/annual_report_pdf_builder.py) |
| section_heading | [annual_report_pdf_builder.py](../../backend/app/annual_reports/services/annual_report_pdf_builder.py) |
| generate | [annual_report_pdf_service.py](../../backend/app/annual_reports/services/annual_report_pdf_service.py) |
| assert_report_exists | [annual_report_service.py](../../backend/app/annual_reports/services/annual_report_service.py) |
| delete_report | [annual_report_service.py](../../backend/app/annual_reports/services/annual_report_service.py) |
| cancel_open_by_client_record | [client_status_service.py](../../backend/app/annual_reports/services/client_status_service.py) |
| create_report | [create_service.py](../../backend/app/annual_reports/services/create_service.py) |
| standard_deadline | [deadlines.py](../../backend/app/annual_reports/services/deadlines.py) |
| extended_deadline | [deadlines.py](../../backend/app/annual_reports/services/deadlines.py) |
| get_detail | [detail_service.py](../../backend/app/annual_reports/services/detail_service.py) |
| update_detail | [detail_service.py](../../backend/app/annual_reports/services/detail_service.py) |
| audit_scalar | [financial_line_helpers.py](../../backend/app/annual_reports/services/financial_line_helpers.py) |
| income_line_snapshot | [financial_line_helpers.py](../../backend/app/annual_reports/services/financial_line_helpers.py) |
| expense_line_snapshot | [financial_line_helpers.py](../../backend/app/annual_reports/services/financial_line_helpers.py) |
| assert_client_allows_financial_mutation | [financial_line_helpers.py](../../backend/app/annual_reports/services/financial_line_helpers.py) |
| add_income | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| update_income | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| delete_income | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| add_expense | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| update_expense | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| delete_expense | [financial_line_service.py](../../backend/app/annual_reports/services/financial_line_service.py) |
| get_financial_summary | [financial_summary_service.py](../../backend/app/annual_reports/services/financial_summary_service.py) |
| get_financial_summary_for_report | [financial_summary_service.py](../../backend/app/annual_reports/services/financial_summary_service.py) |
| calculate_national_insurance | [ni_engine.py](../../backend/app/annual_reports/services/ni_engine.py) |
| get_report | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_client_reports | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| list_reports | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_season_summary | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_overdue | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_status_history | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_detail_report | [query_service.py](../../backend/app/annual_reports/services/query_service.py) |
| get_readiness_check | [readiness_service.py](../../backend/app/annual_reports/services/readiness_service.py) |
| add_schedule | [schedule_service.py](../../backend/app/annual_reports/services/schedule_service.py) |
| complete_schedule | [schedule_service.py](../../backend/app/annual_reports/services/schedule_service.py) |
| get_schedules | [schedule_service.py](../../backend/app/annual_reports/services/schedule_service.py) |
| schedules_complete | [schedule_service.py](../../backend/app/annual_reports/services/schedule_service.py) |
| get_active_annual_report_tax_year | [season_service.py](../../backend/app/annual_reports/services/season_service.py) |
| get_filing_season_year | [season_service.py](../../backend/app/annual_reports/services/season_service.py) |
| get_default_tax_year_response | [season_service.py](../../backend/app/annual_reports/services/season_service.py) |
| get_season_summary_response | [season_service.py](../../backend/app/annual_reports/services/season_service.py) |
| transition_status | [status_service.py](../../backend/app/annual_reports/services/status_service.py) |
| update_deadline | [status_service.py](../../backend/app/annual_reports/services/status_service.py) |
| transition_stage | [status_service.py](../../backend/app/annual_reports/services/status_service.py) |
| amend_report | [status_service.py](../../backend/app/annual_reports/services/status_service.py) |
| calculate_tax | [tax_engine.py](../../backend/app/annual_reports/services/tax_engine.py) |
| get_tax_calculation | [tax_service.py](../../backend/app/annual_reports/services/tax_service.py) |
| get_tax_calculation_for_report | [tax_service.py](../../backend/app/annual_reports/services/tax_service.py) |
| save_tax_calculation | [tax_service.py](../../backend/app/annual_reports/services/tax_service.py) |
| invalidate_tax_if_open | [tax_service.py](../../backend/app/annual_reports/services/tax_service.py) |
| auto_populate | [vat_import_service.py](../../backend/app/annual_reports/services/vat_import_service.py) |

## Repository Functions
| Function | File |
|---|---|
| list_by_report_and_schedule | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| get_schedule_entry | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| get_or_create_schedule_entry | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| next_line_number | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| add_line | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| get_by_id | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| update_line | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| delete_line | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| count_by_report_and_schedule | [annex_data_repository.py](../../backend/app/annual_reports/repositories/annex_data_repository.py) |
| list_by_report_id | [credit_point_repository.py](../../backend/app/annual_reports/repositories/credit_point_repository.py) |
| create | [credit_point_repository.py](../../backend/app/annual_reports/repositories/credit_point_repository.py) |
| aggregate_breakdown | [credit_point_repository.py](../../backend/app/annual_reports/repositories/credit_point_repository.py) |
| total_points_by_report_id | [credit_point_repository.py](../../backend/app/annual_reports/repositories/credit_point_repository.py) |
| get_by_report_id | [detail_repository.py](../../backend/app/annual_reports/repositories/detail_repository.py) |
| update_meta | [detail_repository.py](../../backend/app/annual_reports/repositories/detail_repository.py) |
| create_for_report | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| list_by_report | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| get_by_report_and_line_id | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| apply_updates | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| delete_line | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| total_expenses | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| total_recognized_expenses | [expense_repository.py](../../backend/app/annual_reports/repositories/expense_repository.py) |
| create_for_report | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| list_by_report | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| get_by_report_and_line_id | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| apply_updates | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| delete_line | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| total_income | [income_repository.py](../../backend/app/annual_reports/repositories/income_repository.py) |
| list_overdue | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| count_overdue | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| sum_financials_by_year | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| list_stuck_reports | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| list_for_dashboard | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| get_default_tax_year_candidate | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| get_latest_tax_year | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| get_season_summary | [report_lifecycle_repository.py](../../backend/app/annual_reports/repositories/report_lifecycle_repository.py) |
| create | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| list_by_client_record | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| count_by_client_record | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| get_by_client_record_year | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| list_by_status | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| count_by_status | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| list_by_tax_year | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| count_by_tax_year | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| list_all | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| count_all | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| list_by_tax_year_with_client | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| update | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| soft_delete | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| cancel_open_by_client_record | [report_repository.py](../../backend/app/annual_reports/repositories/report_repository.py) |
| add_schedule | [schedule_repository.py](../../backend/app/annual_reports/repositories/schedule_repository.py) |
| get_schedules | [schedule_repository.py](../../backend/app/annual_reports/repositories/schedule_repository.py) |
| mark_schedule_complete | [schedule_repository.py](../../backend/app/annual_reports/repositories/schedule_repository.py) |
| schedules_complete | [schedule_repository.py](../../backend/app/annual_reports/repositories/schedule_repository.py) |
| append_status_history | [status_history_repository.py](../../backend/app/annual_reports/repositories/status_history_repository.py) |
| get_status_history | [status_history_repository.py](../../backend/app/annual_reports/repositories/status_history_repository.py) |

---

# audit

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{entity_type}/{entity_id} | get_entity_audit_trail | [routes.py](../../backend/app/audit/api/routes.py) |

## Service Functions
| Function | File |
|---|---|
| get_entity_audit_trail | [audit_trail_service.py](../../backend/app/audit/services/audit_trail_service.py) |
| append | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |
| record_create | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |
| record_update | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |
| record_delete | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |
| record_restore | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |
| record_status_change | [entity_audit_writer.py](../../backend/app/audit/services/entity_audit_writer.py) |

## Repository Functions
| Function | File |
|---|---|
| append | [entity_audit_log_repository.py](../../backend/app/audit/repositories/entity_audit_log_repository.py) |
| get_audit_trail | [entity_audit_log_repository.py](../../backend/app/audit/repositories/entity_audit_log_repository.py) |
| count_audit_trail | [entity_audit_log_repository.py](../../backend/app/audit/repositories/entity_audit_log_repository.py) |
| list_recent | [entity_audit_log_repository.py](../../backend/app/audit/repositories/entity_audit_log_repository.py) |

---

# authority_contacts

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | /{client_record_id}/authority-contacts | create_authority_contact | [authority_contact.py](../../backend/app/authority_contacts/api/authority_contact.py) |
| GET | /{client_record_id}/authority-contacts | list_authority_contacts | [authority_contact.py](../../backend/app/authority_contacts/api/authority_contact.py) |
| GET | /{client_record_id}/authority-contacts/{contact_id} | get_authority_contact | [authority_contact.py](../../backend/app/authority_contacts/api/authority_contact.py) |
| PATCH | /{client_record_id}/authority-contacts/{contact_id} | update_authority_contact | [authority_contact.py](../../backend/app/authority_contacts/api/authority_contact.py) |
| DELETE | /{client_record_id}/authority-contacts/{contact_id} | delete_authority_contact | [authority_contact.py](../../backend/app/authority_contacts/api/authority_contact.py) |

## Service Functions
| Function | File |
|---|---|
| add_contact | [authority_contact_service.py](../../backend/app/authority_contacts/services/authority_contact_service.py) |
| update_contact | [authority_contact_service.py](../../backend/app/authority_contacts/services/authority_contact_service.py) |
| list_client_contacts | [authority_contact_service.py](../../backend/app/authority_contacts/services/authority_contact_service.py) |
| delete_contact | [authority_contact_service.py](../../backend/app/authority_contacts/services/authority_contact_service.py) |
| get_contact | [authority_contact_service.py](../../backend/app/authority_contacts/services/authority_contact_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |
| list_by_client_record | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |
| count_by_client_record | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |
| get_for_client | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |
| update_for_client | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |
| delete_for_client | [authority_contact_repository.py](../../backend/app/authority_contacts/repositories/authority_contact_repository.py) |

---

# binders

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{binder_id}/history | get_binder_history | [binders_history.py](../../backend/app/binders/api/binders_history.py) |
| GET | /{binder_id}/intakes | get_binder_intakes | [binders_history.py](../../backend/app/binders/api/binders_history.py) |
| PATCH | /{binder_id}/intakes/{intake_id} | patch_binder_intake | [binders_history.py](../../backend/app/binders/api/binders_history.py) |
| GET | (root) | list_binders | [binders_list_get.py](../../backend/app/binders/api/binders_list_get.py) |
| GET | /{binder_id} | get_binder | [binders_list_get.py](../../backend/app/binders/api/binders_list_get.py) |
| DELETE | /{binder_id} | delete_binder | [binders_list_get.py](../../backend/app/binders/api/binders_list_get.py) |
| GET | /open | list_open_binders | [binders_operations.py](../../backend/app/binders/api/binders_operations.py) |
| POST | /receive | receive_binder | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /mark-ready-for-handover-bulk | mark_ready_for_handover_bulk | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /handover-to-client-bulk | handover_to_client_bulk | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/receive-material | receive_material | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/mark-full | mark_full | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/reopen-capacity | reopen_capacity | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/mark-ready-for-handover | mark_ready_for_handover | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/revert-ready-for-handover | revert_ready_for_handover | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| POST | /{binder_id}/handover-to-client | handover_to_client | [binders_receive_return.py](../../backend/app/binders/api/binders_receive_return.py) |
| GET | /{client_record_id}/binders | list_client_binders | [client_binders_router.py](../../backend/app/binders/api/client_binders_router.py) |

## Service Functions
| Function | File |
|---|---|
| create_handover | [binder_handover_service.py](../../backend/app/binders/services/binder_handover_service.py) |
| build_history_entries | [binder_history_service.py](../../backend/app/binders/services/binder_history_service.py) |
| get_binder_history | [binder_history_service.py](../../backend/app/binders/services/binder_history_service.py) |
| get_binder_intakes | [binder_history_service.py](../../backend/app/binders/services/binder_history_service.py) |
| edit_intake | [binder_intake_edit_service.py](../../backend/app/binders/services/binder_intake_edit_service.py) |
| receive | [binder_intake_service.py](../../backend/app/binders/services/binder_intake_service.py) |
| get_available_action_keys | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| get_available_action_keys_for_state | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| is_intake_eligible | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| receive_material | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| receive_material_by_id | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| mark_full | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| reopen_capacity | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| mark_ready_for_handover | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| mark_ready_for_handover_bulk | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| revert_ready_for_handover | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| handover_to_client | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| handover_loaded_binder | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| log_initial_state | [binder_lifecycle_service.py](../../backend/app/binders/services/binder_lifecycle_service.py) |
| build_binder_response | [binder_list_service.py](../../backend/app/binders/services/binder_list_service.py) |
| get_binder_with_client_name | [binder_list_service.py](../../backend/app/binders/services/binder_list_service.py) |
| list_binders_enriched | [binder_list_service.py](../../backend/app/binders/services/binder_list_service.py) |
| get_open_binders | [binder_operations_service.py](../../backend/app/binders/services/binder_operations_service.py) |
| get_client_binders | [binder_operations_service.py](../../backend/app/binders/services/binder_operations_service.py) |
| get_active_binder_for_client | [binder_operations_service.py](../../backend/app/binders/services/binder_operations_service.py) |
| map_active_binders_for_clients | [binder_operations_service.py](../../backend/app/binders/services/binder_operations_service.py) |
| enrich_binder | [binder_operations_service.py](../../backend/app/binders/services/binder_operations_service.py) |
| receive_binder | [binder_service.py](../../backend/app/binders/services/binder_service.py) |
| delete_binder | [binder_service.py](../../backend/app/binders/services/binder_service.py) |
| create_initial_binder | [client_onboarding_service.py](../../backend/app/binders/services/client_onboarding_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [binder_handover_repository.py](../../backend/app/binders/repositories/binder_handover_repository.py) |
| get_by_id | [binder_handover_repository.py](../../backend/app/binders/repositories/binder_handover_repository.py) |
| list_by_client_record | [binder_handover_repository.py](../../backend/app/binders/repositories/binder_handover_repository.py) |
| get_binder_ids_for_handover | [binder_handover_repository.py](../../backend/app/binders/repositories/binder_handover_repository.py) |
| append | [binder_intake_edit_log_repository.py](../../backend/app/binders/repositories/binder_intake_edit_log_repository.py) |
| list_by_intake | [binder_intake_edit_log_repository.py](../../backend/app/binders/repositories/binder_intake_edit_log_repository.py) |
| create | [binder_intake_material_repository.py](../../backend/app/binders/repositories/binder_intake_material_repository.py) |
| list_by_intake | [binder_intake_material_repository.py](../../backend/app/binders/repositories/binder_intake_material_repository.py) |
| list_by_binder | [binder_intake_material_repository.py](../../backend/app/binders/repositories/binder_intake_material_repository.py) |
| get_last_by_binder | [binder_intake_material_repository.py](../../backend/app/binders/repositories/binder_intake_material_repository.py) |
| create | [binder_intake_repository.py](../../backend/app/binders/repositories/binder_intake_repository.py) |
| get_by_id | [binder_intake_repository.py](../../backend/app/binders/repositories/binder_intake_repository.py) |
| get_first_by_binder | [binder_intake_repository.py](../../backend/app/binders/repositories/binder_intake_repository.py) |
| list_by_binder_page | [binder_intake_repository.py](../../backend/app/binders/repositories/binder_intake_repository.py) |
| append | [binder_lifecycle_log_repository.py](../../backend/app/binders/repositories/binder_lifecycle_log_repository.py) |
| list_all_by_binder | [binder_lifecycle_log_repository.py](../../backend/app/binders/repositories/binder_lifecycle_log_repository.py) |
| list_by_binder_page | [binder_lifecycle_log_repository.py](../../backend/app/binders/repositories/binder_lifecycle_log_repository.py) |
| list_recent | [binder_lifecycle_log_repository.py](../../backend/app/binders/repositories/binder_lifecycle_log_repository.py) |
| create | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| get_active_by_number | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_active | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_active_paginated | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_active_paginated_projected | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_by_lifecycle_filtered | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_active | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_open_binders | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_open_binders | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_by_client_record | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_by_client_record_paginated | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_by_client_record | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| get_active_by_client_record | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_by_client_paginated | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_by_client | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| count_all_by_client | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| map_active_by_clients | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| list_overdue_handover | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| close_in_office_by_client_record | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |
| soft_delete | [binder_repository.py](../../backend/app/binders/repositories/binder_repository.py) |

---

# businesses

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{client_id}/status-card | get_client_status_card | [client_status_card_router.py](../../backend/app/businesses/api/client_status_card_router.py) |
| POST | (root) | create_business | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |
| GET | (root) | list_client_businesses | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |
| GET | /{business_id} | get_business | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |
| PATCH | /{business_id} | update_business | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |
| DELETE | /{business_id} | delete_business | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |
| POST | /{business_id}/restore | restore_business | [client_businesses_router.py](../../backend/app/businesses/api/client_businesses_router.py) |

## Service Functions
| Function | File |
|---|---|
| contact_email | [business_contact_service.py](../../backend/app/businesses/services/business_contact_service.py) |
| contact_phone | [business_contact_service.py](../../backend/app/businesses/services/business_contact_service.py) |
| display_name | [business_contact_service.py](../../backend/app/businesses/services/business_contact_service.py) |
| get_business_or_raise | [business_guards.py](../../backend/app/businesses/services/business_guards.py) |
| assert_business_allows_create | [business_guards.py](../../backend/app/businesses/services/business_guards.py) |
| validate_business_for_create | [business_guards.py](../../backend/app/businesses/services/business_guards.py) |
| assert_business_belongs_to_legal_entity | [business_guards.py](../../backend/app/businesses/services/business_guards.py) |
| delete_business | [business_lifecycle_service.py](../../backend/app/businesses/services/business_lifecycle_service.py) |
| restore_business | [business_lifecycle_service.py](../../backend/app/businesses/services/business_lifecycle_service.py) |
| create_business | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| create_business_for_client_record | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| get_business | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| get_business_or_raise | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| list_businesses_for_client | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| update_business | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| delete_business | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| restore_business | [business_service.py](../../backend/app/businesses/services/business_service.py) |
| to_response | [client_business_service.py](../../backend/app/businesses/services/client_business_service.py) |
| list_for_client | [client_business_service.py](../../backend/app/businesses/services/client_business_service.py) |
| get_for_client | [client_business_service.py](../../backend/app/businesses/services/client_business_service.py) |
| delete_for_client | [client_business_service.py](../../backend/app/businesses/services/client_business_service.py) |
| restore_for_client | [client_business_service.py](../../backend/app/businesses/services/client_business_service.py) |
| compute_business_operational_signals | [signals_service.py](../../backend/app/businesses/services/signals_service.py) |
| get_status_card | [status_card_service.py](../../backend/app/businesses/services/status_card_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| update | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| soft_delete | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| restore | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| get_by_id_including_deleted | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| exists_for_legal_entity | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| all_non_deleted_are_closed_for_legal_entity | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| get_ids_by_legal_entity | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| has_conflicting_sole_trader | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| count | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list_by_legal_entity | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| count_by_legal_entity | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list_by_legal_entity_including_deleted | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list_by_legal_entity_ids | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list_by_ids | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |
| list_all | [business_repository.py](../../backend/app/businesses/repositories/business_repository.py) |

---

# charges

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | (root) | create_charge | [charge.py](../../backend/app/charges/api/charge.py) |
| POST | /{charge_id}/issue | issue_charge | [charge.py](../../backend/app/charges/api/charge.py) |
| POST | /{charge_id}/mark-paid | mark_charge_paid | [charge.py](../../backend/app/charges/api/charge.py) |
| POST | /{charge_id}/cancel | cancel_charge | [charge.py](../../backend/app/charges/api/charge.py) |
| GET | (root) | list_charges | [charge.py](../../backend/app/charges/api/charge.py) |
| GET | /{charge_id} | get_charge | [charge.py](../../backend/app/charges/api/charge.py) |
| POST | /bulk-action | bulk_charge_action | [charge.py](../../backend/app/charges/api/charge.py) |
| DELETE | /{charge_id} | delete_charge | [charge.py](../../backend/app/charges/api/charge.py) |

## Service Functions
| Function | File |
|---|---|
| record_charge_status_audit | [billing_audit.py](../../backend/app/charges/services/billing_audit.py) |
| create_charge | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| issue_charge | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| mark_charge_paid | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| cancel_charge | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| delete_charge | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| get_charge | [billing_service.py](../../backend/app/charges/services/billing_service.py) |
| bulk_action | [bulk_billing_service.py](../../backend/app/charges/services/bulk_billing_service.py) |
| enrich_charge_context | [charge_query_service.py](../../backend/app/charges/services/charge_query_service.py) |
| list_charges | [charge_query_service.py](../../backend/app/charges/services/charge_query_service.py) |
| list_charges_paginated | [charge_query_service.py](../../backend/app/charges/services/charge_query_service.py) |
| build | [charge_response_builder.py](../../backend/app/charges/services/charge_response_builder.py) |

## Repository Functions
| Function | File |
|---|---|
| list_by_annual_report | [charge_annual_report_repository.py](../../backend/app/charges/repositories/charge_annual_report_repository.py) |
| create | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| list_charges | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| list_charges_by_client_record | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| count_charges | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| count_charges_by_client_record | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| sum_open_charges_amount | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| open_charges_stats | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| update_status | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| stats_by_status | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| get_aging_buckets_paginated | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| get_aging_totals | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |
| soft_delete | [charge_repository.py](../../backend/app/charges/repositories/charge_repository.py) |

---

# clients

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | /preview-impact | preview_creation_impact | [clients.py](../../backend/app/clients/api/clients.py) |
| POST | (root) | create_client | [clients.py](../../backend/app/clients/api/clients.py) |
| GET | (root) | list_clients | [clients.py](../../backend/app/clients/api/clients.py) |
| GET | /sidebar | list_sidebar_clients | [clients.py](../../backend/app/clients/api/clients.py) |
| GET | /{client_id} | get_client | [clients.py](../../backend/app/clients/api/clients.py) |
| GET | /conflict/{id_number} | get_conflict_info | [clients.py](../../backend/app/clients/api/clients.py) |
| PATCH | /{client_id} | update_client | [clients.py](../../backend/app/clients/api/clients.py) |
| DELETE | /{client_id} | delete_client | [clients.py](../../backend/app/clients/api/clients.py) |
| POST | /{client_id}/restore | restore_client | [clients.py](../../backend/app/clients/api/clients.py) |
| GET | /export | export_clients | [clients_excel.py](../../backend/app/clients/api/clients_excel.py) |
| GET | /template | download_client_template | [clients_excel.py](../../backend/app/clients/api/clients_excel.py) |
| POST | /import | import_clients_from_excel | [clients_excel.py](../../backend/app/clients/api/clients_excel.py) |

## Service Functions
| Function | File |
|---|---|
| enrich_single | [client_enrichment_service.py](../../backend/app/clients/services/client_enrichment_service.py) |
| enrich_list | [client_enrichment_service.py](../../backend/app/clients/services/client_enrichment_service.py) |
| export_clients | [client_excel_service.py](../../backend/app/clients/services/client_excel_service.py) |
| generate_template | [client_excel_service.py](../../backend/app/clients/services/client_excel_service.py) |
| import_clients_from_excel | [client_excel_service.py](../../backend/app/clients/services/client_excel_service.py) |
| import_clients_from_upload | [client_excel_service.py](../../backend/app/clients/services/client_excel_service.py) |
| delete_client | [client_lifecycle_service.py](../../backend/app/clients/services/client_lifecycle_service.py) |
| restore_client | [client_lifecycle_service.py](../../backend/app/clients/services/client_lifecycle_service.py) |
| run | [client_onboarding_orchestrator.py](../../backend/app/clients/services/client_onboarding_orchestrator.py) |
| get_client_stats | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| list_all_clients | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| get_conflict_info | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| get_full_client | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| get_full_client_including_deleted | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| list_full_clients | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| list_sidebar_clients | [client_query_service.py](../../backend/app/clients/services/client_query_service.py) |
| get_client_or_raise | [client_service.py](../../backend/app/clients/services/client_service.py) |
| update_client | [client_update_service.py](../../backend/app/clients/services/client_update_service.py) |
| create_client | [create_client_service.py](../../backend/app/clients/services/create_client_service.py) |
| create_from_request | [create_client_service.py](../../backend/app/clients/services/create_client_service.py) |
| create_client_identity_only | [create_client_service.py](../../backend/app/clients/services/create_client_service.py) |
| compute_creation_impact | [impact_preview_service.py](../../backend/app/clients/services/impact_preview_service.py) |

## Repository Functions
| Function | File |
|---|---|
| scope_to_active_clients_stmt | [active_client_scope.py](../../backend/app/clients/repositories/active_client_scope.py) |
| apply_graph_update | [client_graph_writer.py](../../backend/app/clients/repositories/client_graph_writer.py) |
| get_display_map | [client_identity_repository.py](../../backend/app/clients/repositories/client_identity_repository.py) |
| get_full_record | [client_record_read_repository.py](../../backend/app/clients/repositories/client_record_read_repository.py) |
| get_full_record_including_deleted | [client_record_read_repository.py](../../backend/app/clients/repositories/client_record_read_repository.py) |
| get_full_records_bulk | [client_record_read_repository.py](../../backend/app/clients/repositories/client_record_read_repository.py) |
| create | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| update_status | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| soft_delete | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| restore | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_by_id | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_by_id_including_deleted | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_by_legal_entity_id | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_legal_entity_id_by_client_record_id | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_active_by_id_number | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_deleted_by_id_number | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| list_by_ids | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| list | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| list_sidebar | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| count | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| count_sidebar | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| search | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| list_all | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| get_signer_name_by_legal_entity_id | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| count_by_status | [client_record_repository.py](../../backend/app/clients/repositories/client_record_repository.py) |
| count_active_by_vat_type | [client_vat_stats_repository.py](../../backend/app/clients/repositories/client_vat_stats_repository.py) |
| count_active_by_vat_types | [client_vat_stats_repository.py](../../backend/app/clients/repositories/client_vat_stats_repository.py) |
| count_active_by_entity_and_vat_type | [client_vat_stats_repository.py](../../backend/app/clients/repositories/client_vat_stats_repository.py) |
| count_active_exempt | [client_vat_stats_repository.py](../../backend/app/clients/repositories/client_vat_stats_repository.py) |

---

# communications

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{client_record_id}/correspondence | list_correspondence_by_client | [correspondence.py](../../backend/app/communications/api/correspondence.py) |
| GET | /{client_record_id}/correspondence/{correspondence_id} | get_correspondence | [correspondence.py](../../backend/app/communications/api/correspondence.py) |
| POST | /{client_record_id}/correspondence | create_correspondence | [correspondence.py](../../backend/app/communications/api/correspondence.py) |
| PATCH | /{client_record_id}/correspondence/{correspondence_id} | update_correspondence | [correspondence.py](../../backend/app/communications/api/correspondence.py) |
| DELETE | /{client_record_id}/correspondence/{correspondence_id} | delete_correspondence | [correspondence.py](../../backend/app/communications/api/correspondence.py) |

> Routes are registered on a named router (`@client_router`), not the conventional `@router`.

## Service Functions
| Function | File |
|---|---|
| add_entry | [correspondence_service.py](../../backend/app/communications/services/correspondence_service.py) |
| get_entry | [correspondence_service.py](../../backend/app/communications/services/correspondence_service.py) |
| update_entry | [correspondence_service.py](../../backend/app/communications/services/correspondence_service.py) |
| delete_entry | [correspondence_service.py](../../backend/app/communications/services/correspondence_service.py) |
| list_client_entries | [correspondence_service.py](../../backend/app/communications/services/correspondence_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| list_paginated | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| list_by_client_paginated | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| list_by_client_record_paginated | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| get_by_id | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| update | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |
| soft_delete | [correspondence_repository.py](../../backend/app/communications/repositories/correspondence_repository.py) |

---

# contacts

## API Endpoints
No API layer

> Note: a `contacts/api/` folder exists but contains no `@router` route decorators.

## Service Functions
No service layer

> Note: a `contacts/services/` folder exists but contains no public `def` functions.

## Repository Functions
No repository layer

> Note: a `contacts/repositories/` folder exists but contains no public `def` functions.

---

# dashboard

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /overview | get_dashboard_overview | [dashboard_overview.py](../../backend/app/dashboard/api/dashboard_overview.py) |
| GET | /tax-submissions | get_tax_submission_widget | [dashboard_tax.py](../../backend/app/dashboard/api/dashboard_tax.py) |

## Service Functions
| Function | File |
|---|---|
| get_overview | [dashboard_overview_service.py](../../backend/app/dashboard/services/dashboard_overview_service.py) |
| get_submission_widget_data | [dashboard_tax_service.py](../../backend/app/dashboard/services/dashboard_tax_service.py) |
| build | [recent_activity_service.py](../../backend/app/dashboard/services/recent_activity_service.py) |
| build | [tax_status_stats_service.py](../../backend/app/dashboard/services/tax_status_stats_service.py) |

## Repository Functions
No repository layer

---

# documents

> Implemented only as the `documents/permanent_documents/` subdomain. No top-level documents API/service/repository.

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /client/{client_record_id}/versions | get_document_versions | [permanent_document_actions.py](../../backend/app/documents/permanent_documents/api/permanent_document_actions.py) |
| GET | /annual-report/{report_id} | list_by_annual_report | [permanent_document_actions.py](../../backend/app/documents/permanent_documents/api/permanent_document_actions.py) |
| POST | /upload | upload_permanent_document | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |
| GET | /client/{client_record_id} | list_client_documents | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |
| GET | /client/{client_record_id}/signals | get_operational_signals | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |
| GET | /client/{client_record_id}/{document_id}/download-url | get_download_url | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |
| DELETE | /client/{client_record_id}/{document_id} | delete_document | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |
| PUT | /client/{client_record_id}/{document_id}/replace | replace_document | [permanent_documents.py](../../backend/app/documents/permanent_documents/api/permanent_documents.py) |

## Service Functions
| Function | File |
|---|---|
| get_document_versions | [permanent_document_action_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_action_service.py) |
| list_by_annual_report | [permanent_document_action_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_action_service.py) |
| upload_document | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| get_download_url | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| list_business_documents | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| list_client_documents | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| get_missing_document_types | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| get_operational_signals | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| get_client_operational_signals | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| delete_document | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| replace_document | [permanent_document_service.py](../../backend/app/documents/permanent_documents/services/permanent_document_service.py) |
| build_one | [response_builder.py](../../backend/app/documents/permanent_documents/services/response_builder.py) |
| build_many | [response_builder.py](../../backend/app/documents/permanent_documents/services/response_builder.py) |

## Repository Functions
| Function | File |
|---|---|
| get_latest_version | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| get_all_versions | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| get_all_versions_by_client | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| get_all_versions_by_client_record | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| list_by_annual_report | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| missing_by_type | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| missing_by_client_type | [permanent_document_query_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_query_repository.py) |
| create | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| get_by_id | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| list_by_business | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| list_by_client | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| list_by_client_record_page | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| count_present_by_client_record | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| get_by_id_and_client_record | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| count_by_business | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| search_by_filename | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| search_by_client_record | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |
| search_by_query | [permanent_document_repository.py](../../backend/app/documents/permanent_documents/repositories/permanent_document_repository.py) |

---

# invoices

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | (root) | attach_invoice | [invoices.py](../../backend/app/invoices/api/invoices.py) |
| GET | /charge/{charge_id} | get_charge_invoice | [invoices.py](../../backend/app/invoices/api/invoices.py) |

## Service Functions
| Function | File |
|---|---|
| attach_invoice_to_charge | [invoice_service.py](../../backend/app/invoices/services/invoice_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [invoice_repository.py](../../backend/app/invoices/repositories/invoice_repository.py) |
| get_by_id | [invoice_repository.py](../../backend/app/invoices/repositories/invoice_repository.py) |
| get_by_charge_id | [invoice_repository.py](../../backend/app/invoices/repositories/invoice_repository.py) |
| list_by_charge_ids | [invoice_repository.py](../../backend/app/invoices/repositories/invoice_repository.py) |
| exists_for_charge | [invoice_repository.py](../../backend/app/invoices/repositories/invoice_repository.py) |

---

# legal_entities

## API Endpoints
No API layer

## Service Functions
No service layer

## Repository Functions
| Function | File |
|---|---|
| create | [legal_entity_repository.py](../../backend/app/legal_entities/repositories/legal_entity_repository.py) |
| get_by_id | [legal_entity_repository.py](../../backend/app/legal_entities/repositories/legal_entity_repository.py) |
| list_by_ids | [legal_entity_repository.py](../../backend/app/legal_entities/repositories/legal_entity_repository.py) |
| get_by_id_number | [legal_entity_repository.py](../../backend/app/legal_entities/repositories/legal_entity_repository.py) |
| create | [person_repository.py](../../backend/app/legal_entities/repositories/person_repository.py) |
| get_by_id_number | [person_repository.py](../../backend/app/legal_entities/repositories/person_repository.py) |
| create_link | [person_repository.py](../../backend/app/legal_entities/repositories/person_repository.py) |
| get_owner_for_legal_entity | [person_repository.py](../../backend/app/legal_entities/repositories/person_repository.py) |
| ensure_owner | [person_repository.py](../../backend/app/legal_entities/repositories/person_repository.py) |

---

# notes

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | (root) | list_notes | [business_notes.py](../../backend/app/notes/api/business_notes.py) |
| POST | (root) | add_note | [business_notes.py](../../backend/app/notes/api/business_notes.py) |
| PATCH | /{note_id} | update_note | [business_notes.py](../../backend/app/notes/api/business_notes.py) |
| DELETE | /{note_id} | delete_note | [business_notes.py](../../backend/app/notes/api/business_notes.py) |
| GET | (root) | list_notes | [entity_notes.py](../../backend/app/notes/api/entity_notes.py) |
| POST | (root) | add_note | [entity_notes.py](../../backend/app/notes/api/entity_notes.py) |
| PATCH | /{note_id} | update_note | [entity_notes.py](../../backend/app/notes/api/entity_notes.py) |
| DELETE | /{note_id} | delete_note | [entity_notes.py](../../backend/app/notes/api/entity_notes.py) |

## Service Functions
| Function | File |
|---|---|
| list_notes | [business_note_service.py](../../backend/app/notes/services/business_note_service.py) |
| add_note | [business_note_service.py](../../backend/app/notes/services/business_note_service.py) |
| update_note | [business_note_service.py](../../backend/app/notes/services/business_note_service.py) |
| delete_note | [business_note_service.py](../../backend/app/notes/services/business_note_service.py) |
| list_notes | [entity_note_service.py](../../backend/app/notes/services/entity_note_service.py) |
| add_note | [entity_note_service.py](../../backend/app/notes/services/entity_note_service.py) |
| update_note | [entity_note_service.py](../../backend/app/notes/services/entity_note_service.py) |
| delete_note | [entity_note_service.py](../../backend/app/notes/services/entity_note_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [entity_note_repository.py](../../backend/app/notes/repositories/entity_note_repository.py) |
| list_for_entity | [entity_note_repository.py](../../backend/app/notes/repositories/entity_note_repository.py) |
| get_by_id | [entity_note_repository.py](../../backend/app/notes/repositories/entity_note_repository.py) |
| update | [entity_note_repository.py](../../backend/app/notes/repositories/entity_note_repository.py) |
| soft_delete | [entity_note_repository.py](../../backend/app/notes/repositories/entity_note_repository.py) |

---

# notifications

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | (root) | list_notifications | [notifications.py](../../backend/app/notifications/api/notifications.py) |
| GET | /summary | get_notification_summary | [notifications.py](../../backend/app/notifications/api/notifications.py) |
| POST | /preview | preview_notification | [notifications.py](../../backend/app/notifications/api/notifications.py) |
| POST | /send | send_notification | [notifications.py](../../backend/app/notifications/api/notifications.py) |

## Service Functions
| Function | File |
|---|---|
| auto_send | [notification_auto_send_service.py](../../backend/app/notifications/services/notification_auto_send_service.py) |
| resolve | [notification_context_resolver.py](../../backend/app/notifications/services/notification_context_resolver.py) |
| resolve_person | [notification_context_resolver.py](../../backend/app/notifications/services/notification_context_resolver.py) |
| resolve_client_name | [notification_context_resolver.py](../../backend/app/notifications/services/notification_context_resolver.py) |
| send | [notification_delivery_service.py](../../backend/app/notifications/services/notification_delivery_service.py) |
| can_send | [notification_policy_service.py](../../backend/app/notifications/services/notification_policy_service.py) |
| preview | [notification_send_service.py](../../backend/app/notifications/services/notification_send_service.py) |
| send | [notification_send_service.py](../../backend/app/notifications/services/notification_send_service.py) |
| preview | [notification_service.py](../../backend/app/notifications/services/notification_service.py) |
| send | [notification_service.py](../../backend/app/notifications/services/notification_service.py) |
| list_paginated | [notification_service.py](../../backend/app/notifications/services/notification_service.py) |
| get_summary | [notification_service.py](../../backend/app/notifications/services/notification_service.py) |
| render | [notification_template_renderer.py](../../backend/app/notifications/services/notification_template_renderer.py) |
| build_preview | [notification_template_renderer.py](../../backend/app/notifications/services/notification_template_renderer.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| mark_sent | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| mark_failed | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| get_by_id | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| find_by_idempotency_key | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| list_paginated | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| count_by_status | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| get_last_for_binder_trigger | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| get_last_for_annual_report_trigger | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| get_last_for_entity_trigger | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| get_last_for_signature_trigger | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| latest_by_binder_ids | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |
| latest_by_annual_report_ids | [notification_repository.py](../../backend/app/notifications/repositories/notification_repository.py) |

---

# reminders

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | / | create_reminder | [routes_create.py](../../backend/app/reminders/api/routes_create.py) |
| GET | / | list_reminders | [routes_list.py](../../backend/app/reminders/api/routes_list.py) |
| GET | /{reminder_id:int} | get_reminder | [routes_get.py](../../backend/app/reminders/api/routes_get.py) |
| POST | /{reminder_id:int}/cancel | cancel_reminder | [routes_cancel.py](../../backend/app/reminders/api/routes_cancel.py) |

> Routes are registered on named routers (`@create_router`, `@list_router`, `@get_router`, `@cancel_router`), not the conventional `@router`.

## Service Functions
| Function | File |
|---|---|
| fire_due | [reminder_executor_service.py](../../backend/app/reminders/services/reminder_executor_service.py) |
| create_from_request | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |
| get_reminders | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |
| get_reminder | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |
| cancel_reminder | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |
| to_response | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |
| to_responses | [reminder_service.py](../../backend/app/reminders/services/reminder_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [reminder_repository.py](../../backend/app/reminders/repositories/reminder_repository.py) |
| update_status | [reminder_repository.py](../../backend/app/reminders/repositories/reminder_repository.py) |
| list_by_status | [reminder_repository.py](../../backend/app/reminders/repositories/reminder_repository.py) |
| count_by_status | [reminder_repository.py](../../backend/app/reminders/repositories/reminder_repository.py) |
| list_due_scheduled | [reminder_repository.py](../../backend/app/reminders/repositories/reminder_repository.py) |

---

# reports

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /vat-compliance | get_vat_compliance_report | [reports.py](../../backend/app/reports/api/reports.py) |
| GET | /advance-payments | get_advance_payment_report | [reports.py](../../backend/app/reports/api/reports.py) |
| GET | /annual-reports | get_annual_report_status_report | [reports.py](../../backend/app/reports/api/reports.py) |
| GET | /aging | get_aging_report | [reports.py](../../backend/app/reports/api/reports.py) |
| GET | /aging/export | export_aging_report | [reports.py](../../backend/app/reports/api/reports.py) |

## Service Functions
| Function | File |
|---|---|
| get_collections_report | [advance_payment_report.py](../../backend/app/reports/services/advance_payment_report.py) |
| get_report | [annual_report_status_report.py](../../backend/app/reports/services/annual_report_status_report.py) |
| export_aging_report_to_excel | [export_excel.py](../../backend/app/reports/services/export_excel.py) |
| export_aging_report_to_pdf | [export_pdf.py](../../backend/app/reports/services/export_pdf.py) |
| export_aging_report_to_excel | [export_service.py](../../backend/app/reports/services/export_service.py) |
| export_aging_report_to_pdf | [export_service.py](../../backend/app/reports/services/export_service.py) |
| export_aging_report | [reports_export_service.py](../../backend/app/reports/services/reports_export_service.py) |
| generate_aging_report | [reports_service.py](../../backend/app/reports/services/reports_service.py) |
| get_vat_compliance_report | [vat_compliance_report.py](../../backend/app/reports/services/vat_compliance_report.py) |

## Repository Functions
No repository layer

---

# search

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | (root) | search | [search.py](../../backend/app/search/api/search.py) |

## Service Functions
| Function | File |
|---|---|
| search_documents | [document_search_service.py](../../backend/app/search/services/document_search_service.py) |
| list_client_documents | [document_search_service.py](../../backend/app/search/services/document_search_service.py) |
| search | [search_service.py](../../backend/app/search/services/search_service.py) |

## Repository Functions
No repository layer

---

# signature_requests

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | (root) | create_signature_request | [routes_advisor.py](../../backend/app/signature_requests/api/routes_advisor.py) |
| GET | /pending | list_pending_requests | [routes_advisor.py](../../backend/app/signature_requests/api/routes_advisor.py) |
| GET | /{request_id} | get_signature_request | [routes_advisor.py](../../backend/app/signature_requests/api/routes_advisor.py) |
| GET | /clients/{client_record_id}/signature-requests | list_client_signature_requests | [routes_client.py](../../backend/app/signature_requests/api/routes_client.py) |
| POST | /clients/{client_record_id}/signature-requests/{request_id}/cancel | cancel_client_signature_request | [routes_client.py](../../backend/app/signature_requests/api/routes_client.py) |
| GET | /{token} | signer_view | [routes_signer.py](../../backend/app/signature_requests/api/routes_signer.py) |
| POST | /{token}/approve | signer_approve | [routes_signer.py](../../backend/app/signature_requests/api/routes_signer.py) |
| POST | /{token}/decline | signer_decline | [routes_signer.py](../../backend/app/signature_requests/api/routes_signer.py) |

> Routes are registered on named routers (`@advisor_router`, `@client_router`, `@signer_router`), not the conventional `@router`.

## Service Functions
| Function | File |
|---|---|
| cancel_request | [admin_actions.py](../../backend/app/signature_requests/services/admin_actions.py) |
| expire_overdue_requests | [admin_actions.py](../../backend/app/signature_requests/services/admin_actions.py) |
| create_request | [create_request.py](../../backend/app/signature_requests/services/create_request.py) |
| build | [response_builder.py](../../backend/app/signature_requests/services/response_builder.py) |
| build_list | [response_builder.py](../../backend/app/signature_requests/services/response_builder.py) |
| build_with_audit | [response_builder.py](../../backend/app/signature_requests/services/response_builder.py) |
| build_created | [response_builder.py](../../backend/app/signature_requests/services/response_builder.py) |
| create_request | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| record_view | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| sign_request | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| decline_request | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| cancel_request | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| expire_overdue_requests | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| get_request | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| get_by_token | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| list_client_requests | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| list_pending_requests | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| get_audit_trail | [signature_request_service.py](../../backend/app/signature_requests/services/signature_request_service.py) |
| get_or_raise | [signature_request_validations.py](../../backend/app/signature_requests/services/signature_request_validations.py) |
| get_by_token_or_raise | [signature_request_validations.py](../../backend/app/signature_requests/services/signature_request_validations.py) |
| get_by_token_or_raise_for_update | [signature_request_validations.py](../../backend/app/signature_requests/services/signature_request_validations.py) |
| assert_pending | [signature_request_validations.py](../../backend/app/signature_requests/services/signature_request_validations.py) |
| check_not_expired | [signature_request_validations.py](../../backend/app/signature_requests/services/signature_request_validations.py) |
| record_view | [signer_actions.py](../../backend/app/signature_requests/services/signer_actions.py) |
| sign_request | [signer_actions.py](../../backend/app/signature_requests/services/signer_actions.py) |
| decline_request | [signer_actions.py](../../backend/app/signature_requests/services/signer_actions.py) |

## Repository Functions
| Function | File |
|---|---|
| append_audit_event | [signature_request_audit.py](../../backend/app/signature_requests/repositories/signature_request_audit.py) |
| list_audit_events | [signature_request_audit.py](../../backend/app/signature_requests/repositories/signature_request_audit.py) |
| create_pending | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| get_by_id | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| get_by_token | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| get_pending_by_client_and_id_for_update | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| get_by_token_for_update | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| list_by_client_record | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| count_by_client_record | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| list_by_business | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| count_by_business | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| list_pending | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| count_pending | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| update | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| list_pending_by_annual_report | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |
| list_expired_pending | [signature_request_crud.py](../../backend/app/signature_requests/repositories/signature_request_crud.py) |

---

# tasks

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | (root) | list_tasks | [routes.py](../../backend/app/tasks/api/routes.py) |
| POST | (root) | create_task | [routes.py](../../backend/app/tasks/api/routes.py) |
| GET | /{task_id} | get_task | [routes.py](../../backend/app/tasks/api/routes.py) |
| PATCH | /{task_id} | update_task | [routes.py](../../backend/app/tasks/api/routes.py) |
| POST | /{task_id}/complete | complete_task | [routes.py](../../backend/app/tasks/api/routes.py) |
| POST | /{task_id}/cancel | cancel_task | [routes.py](../../backend/app/tasks/api/routes.py) |
| DELETE | /{task_id} | delete_task | [routes.py](../../backend/app/tasks/api/routes.py) |

## Service Functions
| Function | File |
|---|---|
| source_exists | [source_validator.py](../../backend/app/tasks/services/source_validator.py) |
| create | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| list | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| update | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| complete | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| cancel | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| delete | [task_service.py](../../backend/app/tasks/services/task_service.py) |
| get | [task_service.py](../../backend/app/tasks/services/task_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [task_repository.py](../../backend/app/tasks/repositories/task_repository.py) |
| list_active | [task_repository.py](../../backend/app/tasks/repositories/task_repository.py) |
| list_for_work_queue | [task_repository.py](../../backend/app/tasks/repositories/task_repository.py) |
| list_by_ids | [task_repository.py](../../backend/app/tasks/repositories/task_repository.py) |

---

# tax_calendar

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /groups | list_tax_calendar_groups | [groups.py](../../backend/app/tax_calendar/api/groups.py) |
| GET | /groups/{tax_calendar_entry_id}/items | get_tax_calendar_group_items | [groups.py](../../backend/app/tax_calendar/api/groups.py) |
| GET | /rules | list_deadline_rules | [settings.py](../../backend/app/tax_calendar/api/settings.py) |
| GET | /entries | list_tax_calendar_entries | [settings.py](../../backend/app/tax_calendar/api/settings.py) |
| GET | /summary | get_tax_calendar_summary | [settings.py](../../backend/app/tax_calendar/api/settings.py) |
| POST | /bootstrap | bootstrap_tax_calendar_settings | [settings.py](../../backend/app/tax_calendar/api/settings.py) |

## Service Functions
| Function | File |
|---|---|
| seed_default_deadline_rules | [bootstrap.py](../../backend/app/tax_calendar/services/bootstrap.py) |
| default_year_range | [bootstrap.py](../../backend/app/tax_calendar/services/bootstrap.py) |
| bootstrap_tax_calendar | [bootstrap.py](../../backend/app/tax_calendar/services/bootstrap.py) |
| has_overlapping_rule | [deadline_rule_service.py](../../backend/app/tax_calendar/services/deadline_rule_service.py) |
| find_periodic_entry_id | [entry_lookup.py](../../backend/app/tax_calendar/services/entry_lookup.py) |
| find_periodic_entry | [entry_lookup.py](../../backend/app/tax_calendar/services/entry_lookup.py) |
| find_annual_entry_id | [entry_lookup.py](../../backend/app/tax_calendar/services/entry_lookup.py) |
| get_group_items | [grouped_items_service.py](../../backend/app/tax_calendar/services/grouped_items_service.py) |
| list_groups_paginated | [grouped_service.py](../../backend/app/tax_calendar/services/grouped_service.py) |
| find_active_null_tax_calendar_links | [link_diagnostics.py](../../backend/app/tax_calendar/services/link_diagnostics.py) |
| ensure_periodic_entry | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| get_periodic_entry | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| get_periodic_entries | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| ensure_annual_entry | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| link_vat_work_item | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| link_advance_payment | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| link_annual_report | [materialization_service.py](../../backend/app/tax_calendar/services/materialization_service.py) |
| list_rules | [settings_calendar_service.py](../../backend/app/tax_calendar/services/settings_calendar_service.py) |
| list_entries | [settings_calendar_service.py](../../backend/app/tax_calendar/services/settings_calendar_service.py) |
| get_summary | [settings_calendar_service.py](../../backend/app/tax_calendar/services/settings_calendar_service.py) |
| bootstrap_calendar | [settings_calendar_service.py](../../backend/app/tax_calendar/services/settings_calendar_service.py) |
| periodic_due_date | [tax_calendar_entry_service.py](../../backend/app/tax_calendar/services/tax_calendar_entry_service.py) |
| annual_due_date | [tax_calendar_entry_service.py](../../backend/app/tax_calendar/services/tax_calendar_entry_service.py) |
| generate_for_year | [tax_calendar_entry_service.py](../../backend/app/tax_calendar/services/tax_calendar_entry_service.py) |
| generate_for_year_range | [tax_calendar_entry_service.py](../../backend/app/tax_calendar/services/tax_calendar_entry_service.py) |

## Repository Functions
| Function | File |
|---|---|
| has_open_ended_rule | [deadline_rule_repository.py](../../backend/app/tax_calendar/repositories/deadline_rule_repository.py) |
| list_by_type | [deadline_rule_repository.py](../../backend/app/tax_calendar/repositories/deadline_rule_repository.py) |
| resolve_active_rule | [deadline_rule_repository.py](../../backend/app/tax_calendar/repositories/deadline_rule_repository.py) |
| add | [deadline_rule_repository.py](../../backend/app/tax_calendar/repositories/deadline_rule_repository.py) |
| flush | [deadline_rule_repository.py](../../backend/app/tax_calendar/repositories/deadline_rule_repository.py) |
| find_periodic | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| find_annual_id | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| count_in_year_range | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| load_existing_keys | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| add | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| flush | [entry_repository.py](../../backend/app/tax_calendar/repositories/entry_repository.py) |
| list_entries | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_vat_for_entries | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_advance_for_entries | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_annual_for_entries | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| get_entry | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_vat_items | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_advance_items | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| list_annual_items | [grouped_repository.py](../../backend/app/tax_calendar/repositories/grouped_repository.py) |
| null_link_ids | [link_diagnostics_repository.py](../../backend/app/tax_calendar/repositories/link_diagnostics_repository.py) |
| find_null_calendar_links | [link_diagnostics_repository.py](../../backend/app/tax_calendar/repositories/link_diagnostics_repository.py) |
| collect | [link_diagnostics_repository.py](../../backend/app/tax_calendar/repositories/link_diagnostics_repository.py) |
| list_rules | [settings_repository.py](../../backend/app/tax_calendar/repositories/settings_repository.py) |
| list_entries | [settings_repository.py](../../backend/app/tax_calendar/repositories/settings_repository.py) |
| count_by_year_obligation_months | [settings_repository.py](../../backend/app/tax_calendar/repositories/settings_repository.py) |

---

# timeline

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /{client_record_id}/timeline | get_client_timeline | [timeline.py](../../backend/app/timeline/api/timeline.py) |

## Service Functions
| Function | File |
|---|---|
| binder_received_event | [timeline_binder_event_builders.py](../../backend/app/timeline/services/timeline_binder_event_builders.py) |
| binder_handed_over_event | [timeline_binder_event_builders.py](../../backend/app/timeline/services/timeline_binder_event_builders.py) |
| binder_lifecycle_change_event | [timeline_binder_event_builders.py](../../backend/app/timeline/services/timeline_binder_event_builders.py) |
| charge_created_event | [timeline_charge_event_builders.py](../../backend/app/timeline/services/timeline_charge_event_builders.py) |
| charge_issued_event | [timeline_charge_event_builders.py](../../backend/app/timeline/services/timeline_charge_event_builders.py) |
| charge_paid_event | [timeline_charge_event_builders.py](../../backend/app/timeline/services/timeline_charge_event_builders.py) |
| invoice_attached_event | [timeline_charge_event_builders.py](../../backend/app/timeline/services/timeline_charge_event_builders.py) |
| build_client_events | [timeline_client_aggregator.py](../../backend/app/timeline/services/timeline_client_aggregator.py) |
| client_created_event | [timeline_client_builders.py](../../backend/app/timeline/services/timeline_client_builders.py) |
| document_uploaded_event | [timeline_client_builders.py](../../backend/app/timeline/services/timeline_client_builders.py) |
| signature_request_lifecycle_event | [timeline_client_builders.py](../../backend/app/timeline/services/timeline_client_builders.py) |
| notification_sent_event | [timeline_notification_event_builders.py](../../backend/app/timeline/services/timeline_notification_event_builders.py) |
| notification_failed_event | [timeline_notification_event_builders.py](../../backend/app/timeline/services/timeline_notification_event_builders.py) |
| get_client_timeline | [timeline_service.py](../../backend/app/timeline/services/timeline_service.py) |
| annual_report_status_changed_event | [timeline_tax_builders.py](../../backend/app/timeline/services/timeline_tax_builders.py) |

## Repository Functions
| Function | File |
|---|---|
| get_client_record | [timeline_repository.py](../../backend/app/timeline/repositories/timeline_repository.py) |
| list_permanent_documents | [timeline_repository.py](../../backend/app/timeline/repositories/timeline_repository.py) |
| list_signature_lifecycle_events | [timeline_repository.py](../../backend/app/timeline/repositories/timeline_repository.py) |

---

# users

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| POST | /login | login | [auth.py](../../backend/app/users/api/auth.py) |
| POST | /refresh | refresh | [auth.py](../../backend/app/users/api/auth.py) |
| GET | /me | me | [auth.py](../../backend/app/users/api/auth.py) |
| POST | /logout | logout | [auth.py](../../backend/app/users/api/auth.py) |
| POST | /forgot-password | forgot_password | [password_reset.py](../../backend/app/users/api/password_reset.py) |
| POST | /reset-password | reset_password | [password_reset.py](../../backend/app/users/api/password_reset.py) |
| POST | (root) | create_user | [users.py](../../backend/app/users/api/users.py) |
| GET | (root) | list_users | [users.py](../../backend/app/users/api/users.py) |
| GET | /{user_id} | get_user | [users.py](../../backend/app/users/api/users.py) |
| PATCH | /{user_id} | update_user | [users.py](../../backend/app/users/api/users.py) |
| POST | /{user_id}/activate | activate_user | [users.py](../../backend/app/users/api/users.py) |
| POST | /{user_id}/deactivate | deactivate_user | [users.py](../../backend/app/users/api/users.py) |
| POST | /{user_id}/reset-password | reset_password | [users.py](../../backend/app/users/api/users.py) |
| GET | /audit-logs | list_audit_logs | [users_audit.py](../../backend/app/users/api/users_audit.py) |

## Service Functions
| Function | File |
|---|---|
| log | [audit_log_service.py](../../backend/app/users/services/audit_log_service.py) |
| list_logs | [audit_log_service.py](../../backend/app/users/services/audit_log_service.py) |
| hash_password | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| verify_password | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| authenticate | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| login | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| logout_user | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| issue_auth_bundle | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| refresh_access_token | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| logout_by_refresh_token | [auth_service.py](../../backend/app/users/services/auth_service.py) |
| request_password_reset | [password_reset_service.py](../../backend/app/users/services/password_reset_service.py) |
| reset_password | [password_reset_service.py](../../backend/app/users/services/password_reset_service.py) |
| generate_access_token | [token_service.py](../../backend/app/users/services/token_service.py) |
| generate_refresh_token | [token_service.py](../../backend/app/users/services/token_service.py) |
| decode_access_token | [token_service.py](../../backend/app/users/services/token_service.py) |
| decode_refresh_token | [token_service.py](../../backend/app/users/services/token_service.py) |
| ensure_advisor | [user_management_policies.py](../../backend/app/users/services/user_management_policies.py) |
| validate_password | [user_management_policies.py](../../backend/app/users/services/user_management_policies.py) |
| get_user_or_raise | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| create_user | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| list_users | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| get_user | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| update_user | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| activate_user | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| deactivate_user | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |
| reset_password | [user_management_service.py](../../backend/app/users/services/user_management_service.py) |

## Repository Functions
| Function | File |
|---|---|
| create | [password_reset_token_repository.py](../../backend/app/users/repositories/password_reset_token_repository.py) |
| invalidate_unused_tokens_for_user | [password_reset_token_repository.py](../../backend/app/users/repositories/password_reset_token_repository.py) |
| get_valid_by_token_hash | [password_reset_token_repository.py](../../backend/app/users/repositories/password_reset_token_repository.py) |
| mark_used | [password_reset_token_repository.py](../../backend/app/users/repositories/password_reset_token_repository.py) |
| create | [user_audit_log_repository.py](../../backend/app/users/repositories/user_audit_log_repository.py) |
| list | [user_audit_log_repository.py](../../backend/app/users/repositories/user_audit_log_repository.py) |
| count | [user_audit_log_repository.py](../../backend/app/users/repositories/user_audit_log_repository.py) |
| get_by_id | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| get_auth_subject_by_id | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| get_by_email | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| list | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| count | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| list_by_ids | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| create | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| update_last_login | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| update | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| activate | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| bump_token_version | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| deactivate_and_bump_token | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |
| set_password_and_bump_token | [user_repository.py](../../backend/app/users/repositories/user_repository.py) |

---

# vat

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | /clients/{client_record_id}/summary | get_vat_client_summary | [routes_client_summary.py](../../backend/app/vat/api/routes_client_summary.py) |
| GET | /clients/{client_record_id}/export | export_vat_client | [routes_client_summary.py](../../backend/app/vat/api/routes_client_summary.py) |
| POST | /work-items/{item_id}/invoices | add_invoice | [routes_data_entry.py](../../backend/app/vat/api/routes_data_entry.py) |
| GET | /work-items/{item_id}/invoices | list_invoices | [routes_data_entry.py](../../backend/app/vat/api/routes_data_entry.py) |
| PATCH | /work-items/{item_id}/invoices/{invoice_id} | update_invoice | [routes_data_entry.py](../../backend/app/vat/api/routes_data_entry.py) |
| DELETE | /work-items/{item_id}/invoices/{invoice_id} | delete_invoice | [routes_data_entry.py](../../backend/app/vat/api/routes_data_entry.py) |
| POST | /work-items/{item_id}/file | file_vat_return | [routes_filing.py](../../backend/app/vat/api/routes_filing.py) |
| GET | /work-items/groups | list_work_item_groups | [routes_grouped.py](../../backend/app/vat/api/routes_grouped.py) |
| GET | /work-items/groups/{group_key}/items | list_work_item_group_items | [routes_grouped.py](../../backend/app/vat/api/routes_grouped.py) |
| POST | /work-items | create_work_item | [routes_intake.py](../../backend/app/vat/api/routes_intake.py) |
| POST | /work-items/{item_id}/materials-complete | mark_materials_complete | [routes_intake.py](../../backend/app/vat/api/routes_intake.py) |
| GET | /work-items/lookup | lookup_work_item | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /clients/{client_record_id}/period-options | get_period_options | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /work-items/status-summary | get_status_summary | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /work-items/{item_id} | get_work_item | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /clients/{client_record_id}/work-items | list_client_work_items | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /work-items | list_work_items | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| GET | /work-items/{item_id}/audit | get_audit_trail | [routes_queries.py](../../backend/app/vat/api/routes_queries.py) |
| POST | /work-items/{item_id}/ready-for-review | mark_ready_for_review | [routes_status.py](../../backend/app/vat/api/routes_status.py) |
| POST | /work-items/{item_id}/send-back | send_back_for_correction | [routes_status.py](../../backend/app/vat/api/routes_status.py) |

## Service Functions
| Function | File |
|---|---|
| get_active_client_and_entity | [client_context_service.py](../../backend/app/vat/services/client_context_service.py) |
| cancel_open_by_client_record | [client_status_service.py](../../backend/app/vat/services/client_status_service.py) |
| assert_editable | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| assert_transition_allowed | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| recalculate_totals | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| audit_invoice_snapshot | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| resolve_invoice_derived_fields | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| check_osek_patur_ceiling | [data_entry_common.py](../../backend/app/vat/services/data_entry_common.py) |
| delete_invoice | [data_entry_invoice_delete.py](../../backend/app/vat/services/data_entry_invoice_delete.py) |
| update_invoice | [data_entry_invoice_update.py](../../backend/app/vat/services/data_entry_invoice_update.py) |
| add_invoice | [data_entry_invoices.py](../../backend/app/vat/services/data_entry_invoices.py) |
| mark_ready_for_review | [data_entry_status.py](../../backend/app/vat/services/data_entry_status.py) |
| send_back_for_correction | [data_entry_status.py](../../backend/app/vat/services/data_entry_status.py) |
| file_vat_return | [filing.py](../../backend/app/vat/services/filing.py) |
| create_work_item | [intake.py](../../backend/app/vat/services/intake.py) |
| mark_materials_complete | [intake.py](../../backend/app/vat/services/intake.py) |
| get_period_options | [period_options.py](../../backend/app/vat/services/period_options.py) |
| calculate_vat_amount | [vat_amounts.py](../../backend/app/vat/services/vat_amounts.py) |
| split_gross_amount | [vat_amounts.py](../../backend/app/vat/services/vat_amounts.py) |
| get_client_summary | [vat_client_summary_service.py](../../backend/app/vat/services/vat_client_summary_service.py) |
| export_vat_to_excel | [vat_export_excel.py](../../backend/app/vat/services/vat_export_excel.py) |
| export_vat_to_pdf | [vat_export_pdf.py](../../backend/app/vat/services/vat_export_pdf.py) |
| export_to_excel | [vat_export_service.py](../../backend/app/vat/services/vat_export_service.py) |
| export_to_pdf | [vat_export_service.py](../../backend/app/vat/services/vat_export_service.py) |
| export | [vat_export_service.py](../../backend/app/vat/services/vat_export_service.py) |
| get_groups | [vat_grouped_enrichment.py](../../backend/app/vat/services/vat_grouped_enrichment.py) |
| get_group_items_enriched | [vat_grouped_enrichment.py](../../backend/app/vat/services/vat_grouped_enrichment.py) |
| get_work_item_enriched | [vat_report_enrichment.py](../../backend/app/vat/services/vat_report_enrichment.py) |
| get_client_items_enriched | [vat_report_enrichment.py](../../backend/app/vat/services/vat_report_enrichment.py) |
| get_list_enriched | [vat_report_enrichment.py](../../backend/app/vat/services/vat_report_enrichment.py) |
| get_audit_trail_enriched | [vat_report_enrichment.py](../../backend/app/vat/services/vat_report_enrichment.py) |
| deadline_fields_from_snapshot | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| get_vat_deadline_fields | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| get_work_item | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| list_client_work_items_paginated | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| normalize_lookup_period | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| get_work_item_by_client_period | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| list_work_items_by_status | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| list_all_work_items | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| get_status_summary | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| list_invoices | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| get_audit_trail | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| count_audit_trail | [vat_report_queries.py](../../backend/app/vat/services/vat_report_queries.py) |
| create_work_item | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| mark_materials_complete | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_period_options | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| add_invoice | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| delete_invoice | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| update_invoice | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| mark_ready_for_review | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| send_back_for_correction | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| file_vat_return | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_work_item | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| list_client_work_items_paginated | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| list_work_items_by_status | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| list_all_work_items | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_status_summary | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| list_invoices | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_work_item_by_client_period | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_audit_trail | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_work_item_enriched | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_client_items_enriched | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_list_enriched | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| get_audit_trail_enriched | [vat_report_service.py](../../backend/app/vat/services/vat_report_service.py) |
| resolve_effective_vat_type | [vat_type_resolver.py](../../backend/app/vat/services/vat_type_resolver.py) |

## Repository Functions
| Function | File |
|---|---|
| append | [vat_audit_log_repository.py](../../backend/app/vat/repositories/vat_audit_log_repository.py) |
| count_audit_trail | [vat_audit_log_repository.py](../../backend/app/vat/repositories/vat_audit_log_repository.py) |
| get_audit_trail | [vat_audit_log_repository.py](../../backend/app/vat/repositories/vat_audit_log_repository.py) |
| get_annual_output_vat | [vat_client_summary_repository.py](../../backend/app/vat/repositories/vat_client_summary_repository.py) |
| get_periods_for_client | [vat_client_summary_repository.py](../../backend/app/vat/repositories/vat_client_summary_repository.py) |
| get_annual_turnover | [vat_client_summary_repository.py](../../backend/app/vat/repositories/vat_client_summary_repository.py) |
| get_annual_turnover_by_client_ids | [vat_client_summary_repository.py](../../backend/app/vat/repositories/vat_client_summary_repository.py) |
| get_annual_aggregates | [vat_client_summary_repository.py](../../backend/app/vat/repositories/vat_client_summary_repository.py) |
| get_compliance_aggregates_paginated | [vat_compliance_repository.py](../../backend/app/vat/repositories/vat_compliance_repository.py) |
| get_filed_items_for_clients | [vat_compliance_repository.py](../../backend/app/vat/repositories/vat_compliance_repository.py) |
| get_overdue_unfiled | [vat_compliance_repository.py](../../backend/app/vat/repositories/vat_compliance_repository.py) |
| get_stale_pending | [vat_compliance_repository.py](../../backend/app/vat/repositories/vat_compliance_repository.py) |
| sum_vat_both_types | [vat_invoice_aggregation_repository.py](../../backend/app/vat/repositories/vat_invoice_aggregation_repository.py) |
| sum_net_both_types | [vat_invoice_aggregation_repository.py](../../backend/app/vat/repositories/vat_invoice_aggregation_repository.py) |
| sum_income_net_by_client_year | [vat_invoice_aggregation_repository.py](../../backend/app/vat/repositories/vat_invoice_aggregation_repository.py) |
| sum_expense_net_by_client_year_grouped | [vat_invoice_aggregation_repository.py](../../backend/app/vat/repositories/vat_invoice_aggregation_repository.py) |
| create | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| get_by_number | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| list_by_work_item | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| update | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| sum_vat_both_types | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| sum_net_both_types | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| sum_income_net_by_client_year | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| sum_expense_net_by_client_year_grouped | [vat_invoice_repository.py](../../backend/app/vat/repositories/vat_invoice_repository.py) |
| apply_vat_work_item_filters | [vat_work_item_filters.py](../../backend/app/vat/repositories/vat_work_item_filters.py) |
| list_due_date_groups | [vat_work_item_grouped_repository.py](../../backend/app/vat/repositories/vat_work_item_grouped_repository.py) |
| list_by_due_date_paginated | [vat_work_item_grouped_repository.py](../../backend/app/vat/repositories/vat_work_item_grouped_repository.py) |
| get_by_client_record_period | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_by_client_record | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_by_client_record_paginated | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_by_client_record | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_by_business_activity | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_by_status | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_by_status | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_by_status_summary | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_all | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_all | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_by_period_not_filed | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| sum_net_vat_by_client_record_year | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_not_filed_for_period | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| list_open_up_to_period | [vat_work_item_query_repository.py](../../backend/app/vat/repositories/vat_work_item_query_repository.py) |
| count_filed_by_period_type | [vat_work_item_stats_repository.py](../../backend/app/vat/repositories/vat_work_item_stats_repository.py) |
| count_filed_by_period_types | [vat_work_item_stats_repository.py](../../backend/app/vat/repositories/vat_work_item_stats_repository.py) |
| get_by_client_record_period | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_by_client_record | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_by_client_record_paginated | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_by_client_record | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_by_business_activity | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_by_status | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_by_status | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_by_status_summary | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_all | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_all | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_by_period_not_filed | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| sum_net_vat_by_client_record_year | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_not_filed_for_period | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| list_open_up_to_period | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| create | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| update_status | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| cancel_open_by_client_record | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| update_vat_totals | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| mark_filed | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| append_audit | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| count_audit_trail | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |
| get_audit_trail | [vat_work_item_write_repository.py](../../backend/app/vat/repositories/vat_work_item_write_repository.py) |

---

# work_queue

## API Endpoints
| Method | Path | Function | File |
|---|---|---|---|
| GET | (root) | list_work_queue | [routes.py](../../backend/app/work_queue/api/routes.py) |

## Service Functions
| Function | File |
|---|---|
| source_link_action | [actions.py](../../backend/app/work_queue/services/actions.py) |
| task_actions | [actions.py](../../backend/app/work_queue/services/actions.py) |
| create_linked_task_action | [actions.py](../../backend/app/work_queue/services/actions.py) |
| source_actions | [actions.py](../../backend/app/work_queue/services/actions.py) |
| advance_payment_items | [billing_items.py](../../backend/app/work_queue/services/billing_items.py) |
| charge_items | [billing_items.py](../../backend/app/work_queue/services/billing_items.py) |
| binder_items | [binder_items.py](../../backend/app/work_queue/services/binder_items.py) |
| urgency | [common.py](../../backend/app/work_queue/services/common.py) |
| source_key | [common.py](../../backend/app/work_queue/services/common.py) |
| display_status_label | [common.py](../../backend/app/work_queue/services/common.py) |
| load_client_profiles | [common.py](../../backend/app/work_queue/services/common.py) |
| register_client_id | [common.py](../../backend/app/work_queue/services/common.py) |
| preload_client_identities | [common.py](../../backend/app/work_queue/services/common.py) |
| item | [common.py](../../backend/app/work_queue/services/common.py) |
| attach_client_identity | [common.py](../../backend/app/work_queue/services/common.py) |
| vat_work_item_metadata | [metadata.py](../../backend/app/work_queue/services/metadata.py) |
| annual_report_metadata | [metadata.py](../../backend/app/work_queue/services/metadata.py) |
| advance_payment_metadata | [metadata.py](../../backend/app/work_queue/services/metadata.py) |
| charge_metadata | [metadata.py](../../backend/app/work_queue/services/metadata.py) |
| load_source_states | [source_lookup.py](../../backend/app/work_queue/services/source_lookup.py) |
| task_summary | [task_items.py](../../backend/app/work_queue/services/task_items.py) |
| task_item | [task_items.py](../../backend/app/work_queue/services/task_items.py) |
| vat_work_item_items | [tax_items.py](../../backend/app/work_queue/services/tax_items.py) |
| annual_report_items | [tax_items.py](../../backend/app/work_queue/services/tax_items.py) |
| apply_work_queue_filters | [work_queue_service.py](../../backend/app/work_queue/services/work_queue_service.py) |
| build_work_queue_summary | [work_queue_service.py](../../backend/app/work_queue/services/work_queue_service.py) |
| list_items | [work_queue_service.py](../../backend/app/work_queue/services/work_queue_service.py) |
| list_items_with_total | [work_queue_service.py](../../backend/app/work_queue/services/work_queue_service.py) |

## Repository Functions
No repository layer
