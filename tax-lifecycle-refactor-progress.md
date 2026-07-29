## Scope

This file owns only:

- What the tax-lifecycle refactor has actually shipped, wave by wave.
- Decisions taken while executing, and defects found on the way.
- What is knowingly left open, and which wave closes it.

This file must not contain:

- The plan itself (see `docs/tax-lifecycle-refactor-plan.md`).
- Canonical current-state behaviour (see `docs/domains/{vat,advance-payments,annual-reports}.md`).
- Anything described as done that is not committed and verified.

Source of truth: tracking only — not source of truth for current behaviour.

Last updated: 2026-07-29.

# Tax Lifecycle Refactor — Progress

Executes `docs/tax-lifecycle-refactor-plan.md` (decisions D-1 … D-44). The plan was
re-cut into eleven waves W0–W10; **W0, W1 and W2 have shipped.**

Each wave is a vertical slice: backend + `openapi.json` + `generated.ts` + frontend +
seed + tests land together, so the app runs at every wave boundary. Schema changes
are squashed rather than chained — `scripts/dev/reset_dev_db.py` regenerates one
`initial` migration — which is safe only while the system is pre-production.

## Branches

One branch per wave, stacked, in each of the three repos. **Nothing is pushed.**

```
main → tax-lifecycle/w0-delete-duplication
     → tax-lifecycle/w1-liability-range
     → tax-lifecycle/w2-obligation-status
```

| Repo | W0 | W1 | W2 |
|---|---|---|---|
| `backend` | 6 commits | 3 | 8 |
| `frontend` | 1 | 1 | 4 |
| `docs` | 1 | 1 | 1 |

Current migration: `dc5f03405059_initial`. **The Render database must be reset
manually before the next deploy** — that is true after every squashed wave.

## Verification at the W2 boundary

| Check | Result |
|---|---|
| `pytest -q` (backend, full) | **1804 passed, 1 skipped, 0 failed** |
| `ruff check app tests scripts` | clean |
| `alembic check` | models match schema |
| `scripts/dev/reset_dev_db.py` + seed | runs clean |
| `scripts/tooling/check_contract_sync.py` | in sync |
| `npm run check` (frontend) | **FAILED at knip** — masked at the time by reading the result through a pipe; tests/typecheck/lint/format/arch passed. Fixed in the post-W2 review below |
| Worktrees | 0 uncommitted across all three repos |

An earlier revision of this table claimed the frontend check was green. It was not:
`npm run check 2>&1 | grep …` reads the pipe's exit code, not the gate's. Gates are
now read unpiped.

## Post-W2 review (2026-07-28/29)

A user code-review against the plan found six items the wave's own verification
missed. Resolutions, applied on the W2 branch before W3:

1. **`ClientUpdateRequest` validated PATCH fragments without the persisted record** —
   a one-sided range edit that inverted a persisted range surfaced as the DB
   CheckConstraint's 500, and a VAT range could land on an `osek_patur`.
   `ClientUpdateService` now merges the request over the persisted record and runs
   the same `validate_liability_ranges` on the result, raising
   `CLIENT.LIABILITY_RANGE_INVALID` (400). The check runs only when the PATCH touches
   a range or frequency field.
2. **Advance status bypassed the shared graph** — `_status_after_payment` computed the
   target directly; no `assert_transition_allowed`, no per-step audit. Replaced by
   `_payment_status_steps`, which derives the target, walks `stages_between`, asserts
   each step, and each money write records one `advance_payment.status_changed` audit
   row per stage crossed (new action + write policy). D-8's "turnover becomes known →
   `input_received`" stays unwired until the W6/W7 turnover rework.
3. **DEFECT: annual overdue list had `AWAITING_INPUT` twice and omitted
   `INPUT_RECEIVED`** — mechanical-rename leftover in `_overdue_stmt`; now
   `notin_(RESOLVED_OBLIGATION_STATUSES)` like its sibling methods.
4. **`obligation_resolved_expr()` was never created** — intent already met by the
   shared `RESOLVED_OBLIGATION_STATUSES` frozenset feeding both the Python and SQL
   sides; the named helper is judged unnecessary. No change.
5. **DEFECT: `VatProgressBar` checked the deleted `'filed'` literal** — survived the
   sweep because the prop was `string`-typed. Prop is now `VatWorkItemStatus` and the
   check is `'submitted'`. A stale `collecting_docs` in `json_examples.py` fell to the
   same literal-grep.
6. **`npm run check` failed at knip** on nine dead exports. The dead code was deleted
   (`Divider` primitive, `TAX_CALENDAR_OBLIGATION_TYPES`, `AmendReportModalProps`) and
   types used only in-file were un-exported; knip now exits 0.

---

## W0 — Delete provable duplication

Each rule below had more than one implementation; each now has one.

| Rule | Before | After |
|---|---|---|
| Bi-monthly period alignment | 3 implementations, 3 error codes, plus a 4th Pydantic copy that shadowed them with a 422 | `TaxCalendarMaterializationService` only. `VAT.INVALID_PERIOD_FOR_FREQUENCY` and `ADVANCE_PAYMENT.INVALID_PERIOD` retired |
| Bi-monthly period helper | `bimonthly_vat_period` and `bimonthly_advance_payment_period`, byte-identical | `latest_bimonthly_period` |
| Period parsing | ~12 hand-rolled `int(period[:4])` sites | `parse_period`, which validates and raises `TAX_CALENDAR.INVALID_PERIOD` |
| "Advances paid for client+year" | SQL aggregate **and** a Python sum over a `page_size=10000` read | the SQL aggregate |
| `final_balance` | computed 3 times from 2 data sources | published once by `AnnualReportTaxService` |
| Hebrew month names | 2 maps | `HEBREW_MONTHS` |

**Bug fixed:** VAT's resolved-status set omitted `CANCELED` while its SQL excluded it,
so a cancelled period read **open** on the grouped tax calendar and **closed** on the
compliance list. `get_overdue_unfiled`, the method containing it, had no test coverage
at all.

**Correction to the plan:** `VAT.CLIENT_CLOSED` is *not* dead as §3.4 claims. It is
still raised on invoice create — a fourth fork of the client-eligibility rule,
answering 400 where the shared guard answers 409. Left in place, documented, tracked
for W7. Only `VAT.CLIENT_FROZEN` was genuinely unreferenced.

## W1 — Per-obligation-type liability range (closes O-7, adds D-44)

Six nullable columns on `LegalEntity` — `vat_`/`advance_`/`annual_liable_from|to` —
plus three `CheckConstraint`s guaranteeing each range is orderable.

**Per type, not one client-wide date**, because they move independently: an entity can
register for VAT in June, receive an ITA advance rate in September, and still owe a
full-year annual report for the same year.

**Intersection, not containment.** A bi-monthly period covering May–June on a client
liable from 20 June **is** owed, in full — the authority does not prorate a return.
NULL on either side is unbounded, so an unconfigured client owes what it owed before.

**What it replaced:** `if entry.due_date < reference_date: continue` in the onboarding
sync loops — a *calendar* guard doing a *liability* guard's job, and wrong in both
directions. It created a period the client was not yet liable for whenever that
period's due date had not passed, and it dropped genuinely owed past-due periods for a
client onboarded late. Late clients now receive their past-due obligations; those are
debts, and reconciliation never removes them.

The creation-impact preview carried the same filter and moved with it. It also stopped
materializing tax-calendar entries and no longer takes a DB session — a preview must
not write.

## W2 — One `ObligationStatus`, one transition graph

The largest wave: 122 backend files and 19 frontend files referenced the three status
enums.

### The lifecycle

| # | Value | Label |
|---|---|---|
| 1 | `awaiting_input` | ממתין לחומר |
| 2 | `input_received` | החומר התקבל |
| 3 | `in_progress` | בעבודה |
| 4 | `awaiting_verification` | ממתין לאימות |
| 5 | `submitted` | הוגש |
| — | `canceled` | בוטל (off-ladder) |

Rules live once, in `app/common/obligation_lifecycle.py`: forward one stage at a time;
an event may perform consecutive transitions and records each; backward one stage at a
time **and always with a reason**; `submitted` has no outgoing transition; cancel from
any unlocked stage, and `canceled` is terminal.

### Per domain

- **VAT** — a pure 1:1 rename.
- **Annual reports** — two merges. `not_started` + `collecting_docs` → `awaiting_input`;
  `submitted` + `closed` → `submitted`, so one act now records both the filing and the
  assessment. `input_received` is **empty** for annual reports: it is new behaviour, not
  a rename, and W7 wires the gate.
- **Advance payments** — a derivation, and the domain that changed most. It had no
  lifecycle at all; status was computed from money on every write.

### Deleted, not aliased

Three status enums · both `VALID_TRANSITIONS` tables · `_derive_status` · `ReportStage`
· `STAGE_TO_STATUS` · `ANNUAL_REPORT_FILED_STATUSES` · `get_period_start_months` ·
three Hebrew label maps · two byte-identical frontend variant maps · the tax calendar's
routing table over three per-domain resolved predicates.

---

## Decisions taken while executing

**`status == PAID` was two questions.** They looked like one only because the status was
derived from money — it *literally meant* `paid_amount >= expected_amount`. Once the
status is a real lifecycle they diverge: an advisor may close an underpaid period (the
shortfall is a debt, not an open period), and a period paid in full is not closed until
someone confirms it was reported.

Every site was therefore classified rather than renamed:

- **money** → `paid_in_full_expr()` / `is_paid_in_full`: overview timing filter, batch
  and dashboard completion counts, annual KPIs, `sum_paid_by_client_year`, the bulk
  top-up skip, the bulk turnover-refresh skip.
- **lifecycle** → `ObligationStatus`: work-queue `is_final`, tax-calendar resolved,
  grouped counts.

**No published number changed.** Preserving today's meaning exactly was chosen over
"improving" the semantics mid-conversion.

**Money advances but never locks or rewinds.** `_payment_status_steps` (named
`_status_after_payment` until the post-W2 review): a recorded payment moves a period
to `in_progress`, paid in full moves it to `awaiting_verification`, only a person
moves it to `submitted`, and a terminal record is untouched.

**`/amend` and `/transition` were removed in W2, not in W4/W7 as planned.** Both became
unreachable under the shared graph — reopening a submitted report is forbidden, and
`STAGE_TO_STATUS` mapped to statuses that no longer exist — so they could only ever
return 400. Shipping a permanently-broken endpoint was judged worse than a stated gap.

---

## Defects found and fixed

Four were live bugs in shipped behaviour, not regressions introduced by the refactor:

1. **A cancelled VAT period read open on one screen and closed on another** (W0).
2. **`generate_annual_schedule` ignored the liability range entirely** — it kept its own
   copy of the due-date guard and never consulted `obligation_plan`, building its month
   list from `get_period_start_months`. W1 had fixed only the onboarding path.
3. **Four season-summary counts were permanently zero.** `get_season_summary` keys its
   result by the stored status value, so once the statuses merged, the service's reads
   of the pre-merge names could never match.
4. **The VAT status-summary endpoint returned all zeros.** Same shape: the response
   schema still declared `pending_materials`, `material_received`,
   `data_entry_in_progress`, `ready_for_review`, `filed`.

Plus two that the merge made meaningless rather than wrong:

5. **The dashboard's `reports_not_started`** was derived as `total_clients` minus the
   other buckets, which silently became "clients with no report at all". It now counts
   the `awaiting_input` stage.
6. **Recomputing an expected amount could un-settle a paid advance** — fixed by the
   never-rewind rule.

**Items 2 and 3 were found by a post-wave review, not by the test suite**, and neither
would have failed a test. Grep-verifying a wave's own claims after finishing it is
therefore part of the process, not optional.

## Mistakes made during execution, and their cost

Recorded because the same shapes will recur in later waves:

- **A blanket string replace on the enum names over-matched three times.** It renamed
  seven *class* names that merely start with those strings — including
  `AnnualReportStatusService` and `VatWorkItemStatusSummaryResponse`, which had leaked
  into the OpenAPI schema and so were visible to API consumers. It also flipped a VAT
  test helper's default from `FILED` to `AWAITING_VERIFICATION` (cascading into eight
  turnover failures) and rewrote a notifications query string, though notifications
  have their own enum. **Use a word-boundary-aware replace.**
- **`ruff format app tests` reformats nine files nobody touched** — they sit unformatted
  on `main`. Scope `ruff format` to the paths actually edited.
- **`docker compose restart db_test` destroys the test schema.** That container's data
  is on `tmpfs`. Recovery is drop-schema plus `alembic upgrade head`, never a restart.
- **Concurrent pytest runs share one test database** and produce unreliable totals.
- **`pytest -p no:logging` removes the `caplog` fixture**, manufacturing three errors
  that look real.

---

## Open, deliberately

| Item | Closed by |
|---|---|
| Annual reports have **no amendment path** — the reopen is forbidden by the shared graph and create-amendment does not exist yet | W4 |
| `input_received` is **empty for annual reports** — the gate needs "VAT periods all closed **and** documents received" | W7 |
| `VAT.CLIENT_CLOSED` is a **fourth fork** of the client-eligibility rule, answering 400 where the shared guard answers 409 | W7 |
| The **Render database needs a manual reset** before the next deploy | before deploy |

## Remaining waves

| Wave | Scope | Risk |
|---|---|---|
| **W3** | Closing and locking: full lock on submitted records, record *who* closed it, one shared "what is missing" gate, `closed_late` stored (`NULL`, never `false`, where there is no due date) | low |
| **W4** | Amendment and the uniqueness rule. **The dangerous one** — a mechanism the codebase has never had, and a chain that double-counts produces a wrong number rather than an error, reaching into the annual report's VAT import and the advance turnover lookup | **high** |
| **W5** | Removal and reconciliation | medium |
| **W6** | The deadline shape — contains the one visible product change: VAT periods appear in the work queue before they are late | medium |
| **W7** | Domain surgery — signature flow and turnover layer leave | medium |
| **W8** | Coupling and arithmetic — invert advance→annual behind a port | medium |
| **W9** | Generation and rollover — VAT has no office-wide generation today | medium |
| **W10** | Documentation — new `docs/domains/tax-lifecycle.md` (D-41), archive the plan | none |
