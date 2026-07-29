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

Last updated: 2026-07-29 (W4-pre shipped).

# Tax Lifecycle Refactor — Progress

Executes `docs/tax-lifecycle-refactor-plan.md` (decisions D-1 … D-44). The plan was
re-cut into eleven waves W0–W10; **W0, W1, W2 and W3 have shipped, plus W4-pre —
 an out-of-order slice pulled forward from W7 (see its section for why).**

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
     → tax-lifecycle/w3-closing-locking
     → tax-lifecycle/w4-pre-signature-removal
```

| Repo | W0 | W1 | W2 | W3 | W4-pre |
|---|---|---|---|---|---|
| `backend` | 6 commits | 3 | 8 | 1 | 1 |
| `frontend` | 1 | 1 | 4 | 1 | 1 |
| `docs` | 1 | 1 | 1 | 1 | 1 |

Current migration: `0945ac3465e0_initial`. **The Render database must be reset
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

## W3 — Closing and locking (D-13, D-15, D-16, D-20, D-32)

### The closing facts are one vocabulary

`closed_at` · `closed_by` · `closed_late` now exist on all three domains. VAT renamed
`filed_at`/`filed_by`; annual reports renamed `submitted_at` and gained `closed_by`;
advance payments gained all three **plus `assigned_to`, which it never had** — D-15
requires an assignee before closing and there was nothing to require.

**Decision taken in-session (approved):** the plan's §4.1.7 table kept per-domain
names and only added the missing "by whom". Full unification was chosen instead,
because W3 builds shared closing machinery (one gate, one `closed_late` write, one
audit shape) and the no-alias rule forbids adding `closed_by` beside a live
`filed_by`. The "how" and "reference" facts (`submission_method` /
`submission_reference` / `ita_reference` / `payment_reference`) stay per-domain —
their semantics genuinely differ.

`closed_late` is written once, at the close: the **Israel-local date** of `closed_at`
(new `time_utils.israel_date`, closing the UTC-midnight trap) against the domain's due
date — VAT `due_date_effective`, advance `due_date_effective or due_date`, annual the
`filing_deadline` *as recalculated by the submission itself* (the submission method may
move it). NULL means "no due date at close" (D-32), never False. The advance contract's
computed `paid_late` was **deleted**, not kept as an alias — the stored fact answers
D-20's question ("was it closed late"), and paid-vs-closed diverge under D-16.

### One gate shape

`app/common/obligation_closing.py`: `ClosingReadiness {is_ready, issues}` +
`compute_closed_late` + the shared assignee issue string. All three domains publish it:

- annual `GET /{id}/readiness` — now **five** gates (assignee added; completion /5)
- VAT `GET /work-items/{id}/readiness` — new; assignee + final amount, mirroring the
  file gates exactly
- advance `GET /clients/{cid}/advance-payments/{id}/readiness` — new; assignee +
  turnover known + expected amount computable (rate, override, or hand-entered
  expected). **Payment in full is not a gate** (D-16) — a part-paid period closes.

### Advances can finally be closed

`POST /clients/{cid}/advance-payments/{id}/status` — the advisor route that did not
exist. Single-step through the shared graph (forward, backward-with-reason, cancel);
closing asserts the gate (`ADVANCE_PAYMENT.NOT_READY` with the issue list) and writes
the closing facts. Money still advances stages; only a person closes.

### Full lock (D-13)

- **Annual** — `assert_report_unlocked` on every mutation path: financial lines
  (which previously checked only the *client's* status — the plan's named defect),
  detail, schedules, annexes, credit points, deadline, tax-calculation save,
  reassignment, and soft delete (a closed record is never removed, D-22).
- **Advance** — PATCH, delete, turnover refresh, and transitions out are rejected;
  bulk mark-paid and bulk refresh **skip** closed rows (`"closed"` /
  `skipped_closed`) rather than failing a mixed sweep.
- **VAT** — already behaved this way; unchanged.
- All reuse `ErrorCode.OBLIGATION_LOCKED` + `LOCKED_MESSAGE` from the lifecycle module.

**The notes lock is N/A, not missing:** the survey claimed notes attach to closed
obligations; in fact the entity-notes API serves only `client` entities. The
obligations' own `notes` columns are covered by the domain locks.

### Executing-decisions and gaps closed on the way

- **Annual `assigned_to` was settable only at create** — the new gate would have
  bricked every unassigned report. Added `PATCH /annual-reports/{id}`
  (`assigned_to` only) + an assignee selector in the status panel.
- **System auto-submit records `closed_by = NULL`.** The signature reconciliation
  closes with `changed_by=None` and a system actor on the audit row. **Corrected
  2026-07-29:** this was recorded here as correct-by-design, reasoning that the
  assignee gate still guarantees a named owner. That reasoning does not hold —
  `assigned_to` is *who is responsible*, `closed_by` is *who acted*, and D-13 asks
  for the second. It also re-litigated a question D-5 had already closed: the whole
  auto-submit path was condemned. Resolved by deleting it in **W4-pre**, not by
  amending D-13.
- **The audit write policy was the integration seam:** per-action allowlists rejected
  the new fields (`advance_payment.created`), and the new annual reassign needed an
  `annual_report.updated` policy. The seed run caught both.

### Frontend

Rename sweep (`filed_*`/`submitted_at`/`paid_late` → `closed_*`); shared
`isObligationLocked` + stage helpers beside the W2 vocabulary; annual financial lines
hide add/edit/delete when locked; advance detail locks edit/delete and gains a status
panel (stage forward · close gated on the readiness list · send back with required
note · cancel); annual status panel gains the assignee selector.

### Defects found

1. **Annual demo seed** carried a `status in (SUBMITTED, SUBMITTED)` leftover from the
   W2 merge (was `(SUBMITTED, CLOSED)`).
2. **Seeded submitted rows could lack an assignee**, violating the new invariant —
   seeds now always name one.
3. **`docs/domains/annual-reports.md` still describes pre-W2 statuses** in several
   lifecycle paragraphs (`pending_client`, `in_preparation`, `amend_report`). W3
   updated only the closing/locking content; the rest is recorded doc debt for W10.

### Mistakes made during execution, and their cost

- **The first full suite run failed 601 tests** — the squashed migration added columns
  the shared test database didn't have. Recovery is the documented one (drop schema +
  `alembic upgrade head`), cost one 14-minute run.
- **The W2 pipe mistake was repeated once**: `npm run check … | tail` read the pipe's
  exit code and called a prettier failure green. Caught on the same turn, re-run
  unpiped. The rule stands: gates are read unpiped, no exceptions.

## Verification at the W3 boundary

| Check | Result |
|---|---|
| `pytest -q` (backend, full) | **1815 passed, 10 failed** on the first complete run; all 10 were stale test expectations / new-test setup bugs (no app defects), fixed and their files re-run green (66 tests). A final all-in-one re-run was stopped by request; no app code changed after the full run |
| `ruff check app tests scripts` | clean (post `--fix` on two test-import orderings) |
| `alembic check` | models match schema (`6a293b5c0932_initial`) |
| `scripts/dev/reset_dev_db.py --yes` + seed | runs clean |
| `scripts/tooling/check_contract_sync.py` | in sync |
| `npm run check` (frontend) | exit 0, read unpiped |
| Test DB | schema rebuilt (drop + upgrade head) after the squash |

The 10: two signature auto-submit tests missing the now-required assignee; one
pre-W3 test asserting a mutation on a submitted report merely "does not clear tax"
(it is now forbidden outright — rewritten to assert the lock); the error-doc matrix
needing the four new endpoint rows; and the wave's own new lock test carrying setup
bugs (invalid expense category, wrong route paths, tax result cleared by its own
line mutations).

---

## W4-pre — The annual-report signature flow leaves (D-5)

**Pulled forward out of order, from W7.** Recorded plainly because it breaks the
phase↔wave 1:1 the plan set up.

### Why it moved

W3 shipped one exception to D-13 ("every closed record names its author"): the signature
auto-submit closed a report with `changed_by=None`, so `closed_by` was NULL. The W3 record
justified that as correct-by-design because the assignee gate still guarantees a named
owner. **That justification was wrong** — `assigned_to` answers *who is responsible*,
`closed_by` answers *who acted*; they are different facts. It also missed that D-5 had
already condemned the entire path two days earlier.

So the real choice was never "amend D-13 or not". It was: carry a documented exception for
four waves, or delete the condemned code now. Three things decided it:

1. **W4 is next and is the high-risk wave.** Amendments work directly on the annual status
   transition path — the same code that held the signature branches. Deleting first means
   W4 is written against one less layer instead of routing around machinery due for demolition.
2. **No open design question.** The worry was that `awaiting_verification` would lose its
   meaning for annual reports. It does not: §4.1.1 defines stage 4 generically as
   "ready, awaiting verification", which is exactly VAT's `ready_for_review`. The status is
   shared with VAT and advances and does not move; only the client-signature machinery
   hanging off it goes.
3. **Purely subtractive**, which the plan already rates as P6's lowest-risk half.

### What left

**Annual side** — `annual_report_status_signature_helper.py` (whole file); the
`AWAITING_VERIFICATION` branches in the status service (signature creation, cancel-on-leave,
and the `changed_by is None` guard); `AnnualReportDetail.client_approved_at` (column, DTOs,
meta-column allowlist, audit value-field policy); the readiness gate that read it; and
`ClientRecordRepository.get_signer_name_by_legal_entity_id`, dead once nothing resolves a
signer. Readiness drops **5 → 4** gates and completion is now `passed / 4`.

**Signature side** — `_auto_advance_annual_report`, `reconcile_signed_annual_report_approvals`,
the `sign_request` cross-domain hook (`sign_request` now returns the request, not a tuple),
`list_pending_by_annual_report`, `list_signed_annual_report_approvals_pending_submission`,
the `annual_report_id` FK + its index + relationship, the `signature_request.annual_report_signed`
audit action and its write policy, and the startup + daily reconciliation jobs with their
`lifespan.py` wiring. Two enum values retired: `ANNUAL_REPORT_APPROVAL`, and
`VAT_RETURN_APPROVAL` which was defined and referenced by nothing.

The module itself stays — engagement agreements, powers of attorney, custom documents.

**Frontend** — the signature request type union and label map lose both values; the annual
detail form loses its client-approval date field and becomes notes-only.

### What this buys

`closed_by` on an annual report is **never NULL**. Only an advisor can transition to
`submitted`, so D-13 is an invariant with no exceptions rather than a rule with a documented
hole. The savepoint in the transition path is gone too — it existed only because a signature
in an outer transaction had to survive a failed report transition.

### What it does NOT fix

**Stage 2 (`input_received`) is still empty for annual reports.** Its gate (D-18: VAT periods
all closed **and** documents received) is W7's other half, which this slice deliberately did
not touch. Since the shared graph forbids skipping a stage, reports still step through a stage
that nothing defines. Unchanged by this wave, neither better nor worse.

### Verification

| Check | Result |
|---|---|
| `ruff check app tests scripts` | clean (after `--fix` on unused imports in the six test files this wave touched) |
| `ruff format` | clean; scoped to touched files only, per the standing rule |
| `alembic check` | models match schema (`0945ac3465e0_initial`) |
| `scripts/dev/reset_dev_db.py --yes` + seed | runs clean |
| `scripts/tooling/check_contract_sync.py` | in sync (`openapi.json` −104 lines) |
| frontend `typecheck` / `lint` / `format:check` / `arch:check` / `unused` | all exit 0, each read unpiped |
| **`pytest` / `vitest`** | **NOT RUN — the operator asked for tests to be skipped this wave.** Test *sources* were updated (four dead tests removed, helpers and fixtures rewired) but never executed. This is the one gate W4-pre did not clear. |

Dead tests removed: the signature auto-submit/reconcile pair, the `pending_client`
signature-creation and missing-client-record pair, and the background-job reconciliation test.

### Doc debt closed on the way

`flows/02-annual-report-status-transition.md` was a full wave stale — it documented the
`POST /transition` endpoint and `ReportStage` (both retired in W2 by D-40) and the pre-W2
status names. Rewritten against the shared graph. `docs/domains/annual-reports.md` had the
same pre-W2 staleness flagged in the W3 record; the closing/locking and signature paragraphs
are now correct, but **the remaining pre-W2 status names elsewhere in that file are still
open, still for W10.**

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
| **W4** | Amendment and the uniqueness rule. **The dangerous one** — a mechanism the codebase has never had, and a chain that double-counts produces a wrong number rather than an error, reaching into the annual report's VAT import and the advance turnover lookup | **high** |
| **W5** | Removal and reconciliation | medium |
| **W6** | The deadline shape — contains the one visible product change: VAT periods appear in the work queue before they are late | medium |
| **W7** | Domain surgery — **turnover layer + the D-17 permission split** (the signature half shipped early as W4-pre) | medium |
| **W8** | Coupling and arithmetic — invert advance→annual behind a port | medium |
| **W9** | Generation and rollover — VAT has no office-wide generation today | medium |
| **W10** | Documentation — new `docs/domains/tax-lifecycle.md` (D-41), archive the plan | none |

### Explicitly carried into W7 — D-17, the advance-payments permission split

Recorded here as its own item because in the plan it is **one sentence buried in P6's
turnover paragraph** (§P6, "A secretary may record a payment and move stages (D-17)"),
and nothing else would surface it when W7 opens. Its co-location with the turnover
rework is incidental — the two are unrelated changes that share a paragraph.

**The rule (D-17 / §4.1.9).** A secretary moves an obligation through the working
stages. Advisor-only, in all three domains: **close · send back · cancel · amend ·
delete**. Advance payments are the named outlier — "the advance-payments restriction is
the outlier and retires. Recording a payment that arrived is clerical work, not a
judgement; the judgement is the close, and that stays with the advisor."

**What is still wrong, and where.** Every advance write is advisor-only. Two of them
contradict D-17:

| Route | Today | Under D-17 |
|---|---|---|
| `POST /clients/{cid}/advance-payments/{id}/status` | advisor only | secretary **for forward steps**; advisor for back / cancel / close |
| `POST /advance-payments/bulk-mark-paid` | advisor only | secretary — recording a payment that arrived |

`DELETE /{id}` and `PATCH /{id}` (amend) correctly stay advisor-only. The two
`refresh-turnover` routes retire in the same wave under D-9/D-31, so their role guards
are moot. `POST /generate` and `/bulk-rate-update` are **not classified** by §4.1.9
either way — decide them in W7 rather than assuming.

**It is not a `require_role` swap.** W3 built `POST /status` as one endpoint
multiplexing forward, backward, cancel and close, and the service takes `actor_id` with
no notion of role. D-17 splits authorisation *per transition*, so the route-level guard
cannot express it. **VAT is the model** — separate routes per act, `ready-for-review`
open to advisor **and** secretary, `send-back` advisor only. Recommended: split the
advance endpoint the same way, so the permission is readable from the contract instead
of hidden in a branch.

**Two tests pin the interim state** and must be deleted, not adapted, when this lands.
Both were renamed on 2026-07-29 to say so in their name and docstring, after the
original names (`test_forward_is_advisor_only`,
`test_bulk_mark_paid_forbidden_for_secretary`) were found to assert the target model's
opposite as though it were intended:

- `tests/advance_payments/api/test_advance_payment_status_transitions.py::test_transitions_still_carry_the_blanket_advisor_restriction_pending_d17`
- `tests/advance_payments/api/test_advance_payment_bulk_mark_paid.py::test_bulk_mark_paid_still_advisor_only_pending_d17`

**This is the second instance of the same failure mode** — after W3 recorded
`closed_by = NULL` as correct-by-design when D-5 had already condemned the path. Both
times an interim state was written down as intent. The general rule: **when a wave
leaves something unmigrated, name it as unmigrated in the artefact that asserts it.**
