## Scope

This file owns only:

- The cross-domain refactor plan for the tax lifecycle shared by VAT, advance payments, and annual reports.
- The evidence that motivated it, the decisions taken, and the decisions still open.

This file must not contain:

- Canonical current-state behavior for any of the three domains (see `docs/domains/vat.md`,
  `docs/domains/advance-payments.md`, `docs/domains/annual-reports.md`).
- Architecture rules or endpoint contracts (see `docs/architecture/*`, `docs/backend/architecture.md`).
- Behavior described as if already implemented.

Source of truth: tracking only — not source of truth for current behavior.

Created: 2026-07-27. Status: **planning, not started.** No code changes have been made.

## 0. Deployment context — this drives the ordering

The system is **pre-production**. No obligations exist beyond what the seed builders create; there
are no live clients and no data to preserve. Confirmed with the product owner 2026-07-27.

Two consequences, and they point the same way:

- **The structural phases are cheaper now than they will ever be again.** P5's migrations need
  no backfill and no maintenance window — `reset_dev_db` is sufficient. P7 has no
  backward-compatibility surface. D-3's contract change has no consumers in the wild.
- **P8 is not urgent.** The missing VAT catch-up path (§3.3.2) is a *pre-launch gap*, not a
  live defect: no client is currently missing periods, because no client exists.

**Therefore: do the expensive-later work first, and defer the work that only matters at launch.**
See §7.2 for the resulting order.

---

# Tax Lifecycle Refactor — Plan

## 1. Why

VAT, advance payments, and annual reports are three views of one thing: a client owes a tax
obligation for a period, works it through a lifecycle, and settles it by a deadline. The code does
not say that anywhere. Each domain re-derives the shared concepts with its own vocabulary, its own
error codes, and its own timing rules, so the three drift against each other instead of composing.

The referenced-but-nonexistent "tax-lifecycle plan" in
`backend/app/advance_payments/services/advance_payment_service.py:243` is this file.

## 2. What already works — do not rebuild

A shared spine exists and is half-wired. This refactor **finishes** that pattern; it does not
introduce a new one.

| Shared mechanism | Location | Wired to |
|---|---|---|
| `TaxCalendarEntry` + `TaxCalendarMaterializationService` | `backend/app/tax_calendar/services/tax_calendar_materialization_service.py` | all three (NOT NULL FK on each root) |
| Client-eligibility guard `assert_client_record_is_active` + SQL twin `eligible_client_status_expr` | `backend/app/clients/guards/client_record_guards.py`, `backend/app/clients/repositories/client_active_scope.py` | all three |
| `is_vat_work_item_resolved` / `is_advance_payment_resolved` / `is_annual_report_resolved` | each domain's enum module | `tax_calendar_grouped_service.py` |
| `PeriodicObligationPlan` + per-type plan builders | `backend/app/common/obligation_plan.py` | VAT + advance payments only |
| Keyset-chunked, idempotency-keyed office-wide generation | `backend/app/advance_payments/services/advance_payment_service.py` (`bulk_generate_annual_schedules`) | advance payments only |

The guard/twin pair is the reference shape for every rule this plan consolidates: one Python
implementation, one SQL implementation, documented as required to change together.

## 3. Evidence

### 3.1 The lifecycle is three unrelated shapes

|  | VAT | Advance payments | Annual reports |
|---|---|---|---|
| Unit of work | period `YYYY-MM` | period `YYYY-MM` | `tax_year` |
| Lifecycle | 6-status graph with `VALID_TRANSITIONS` | 3 statuses **derived from money**, no transition graph | 7-status graph with `VALID_TRANSITIONS` |
| Terminal states | `filed`, `canceled` | `paid` | `closed`, `canceled` |
| Deadline fields | `due_date_original` / `due_date_effective` / `due_date_override_reason` | same three **plus** legacy `due_date` | `filing_deadline` + `deadline_type` + `custom_deadline_note` |
| Lateness | see §3.3 — inconsistent | computed `timing_status` | `/overdue` endpoint |
| Who creates rows | client onboarding only | onboarding + per-client `/generate` + office-wide `/bulk-generate` | obligation orchestrator only |

### 3.2 Duplicated rules

1. **Bi-monthly odd-month start rule — three implementations, three error codes.**
   - `backend/app/vat/services/vat_intake_service.py:33-45` → `VAT.INVALID_PERIOD_FOR_FREQUENCY`
   - `backend/app/advance_payments/services/advance_payment_service.py` `_validate_period_months_count` → `ADVANCE_PAYMENT.INVALID_PERIOD`
   - `backend/app/tax_calendar/services/tax_calendar_materialization_service.py:190` `_validate_period_alignment` → `TAX_CALENDAR.INVALID_PERIOD_ALIGNMENT`

   The calendar implementation is the real gate — both domains call it during materialization. The
   other two only fire earlier.

2. **`bimonthly_vat_period` and `bimonthly_advance_payment_period` are byte-identical.**
   `backend/app/common/period_utils.py:43-53` vs `:56-66`.

3. **Period parsing hand-rolled in ~12 call sites** (`int(period[:4])`, `period.split("-")[0]`)
   despite `parse_period_year` / `parse_period_month` existing in the same shared module.
   Sites include `vat_audit.py:16`, `vat_turnover_repository.py:42`,
   `vat_data_entry_invoices_service.py:103,112`, `vat_data_entry_invoice_update_service.py:148,159,162`,
   `vat_data_entry_common.py:116`, `vat_intake_service.py:40`,
   `tax_calendar_materialization_service.py:187`, `tax_calendar_entry.py:178`,
   `work_queue_metadata.py:41`, plus seed builders.

4. **"Advances paid for client + year" has two implementations that can disagree.**
   - SQL aggregate: `advance_payment_aggregation_repository.py:195` `sum_paid_by_client_year`
   - Python sum over a `page_size=10000` read: `annual_report_advances_summary_service.py:41-47`

   Consequence: `final_balance` is computed three different ways —
   `annual_report_query_service.py:150` (`tax_after_credits - advances_paid`),
   `annual_report_advances_summary_service.py:52` (same formula, other data source), and
   `annual_report_tax_service.py:127` (adds NI and VAT balance). The Python sum also silently
   undercounts past the page cap.

5. **The client-eligibility consolidation was never finished in annual reports.**
   `annual_report_create_service.py:61` uses the shared guard, but
   `annual_report_financial_line_helpers.py` `assert_client_allows_financial_mutation` still runs a
   *local* copy of the same rule — raising `ForbiddenError` (403) with `CLIENT.CLOSED` /
   `CLIENT.FROZEN`, the exact codes the shared guard (409, `CLIENT_RECORD.CLOSED`) replaced. Two
   answers to one question inside one domain.

6. **Annual-report financial lines stay editable after submission.** That same helper checks only the
   *client's* status, never the *report's*. Income and expense lines can be added, changed, and
   deleted on a submitted report. VAT locks its invoices on filing; annual reports lock nothing.

7. **Resolved-status set forked into SQL without a twin.**
   `backend/app/vat/repositories/vat_compliance_repository.py:88` hardcodes
   `notin_([FILED, CANCELED])` instead of calling `is_vat_work_item_resolved`. Adding a VAT status
   silently requires editing this repository. This is the exact failure mode the client-eligibility
   guard already solved with `eligible_client_status_expr`.

### 3.3 Flows not aligned

1. **Work-queue entry uses a different rule per domain.**
   - VAT (`backend/app/work_queue/items/tax_items.py` → `VatComplianceRepository.get_overdue_unfiled`)
     filters on **`period < current YYYY-MM`**, not on `due_date_effective`. This contradicts the
     documented invariant that `due_date_effective` is the overdue source of truth, and it means a
     VAT period **never appears in the queue before it is already late** — there is no upcoming
     window.
   - Advance payments and annual reports both use a due-date cutoff of
     `today + UPCOMING_WINDOW_DAYS`.
   - Advance-payment items are assembled in `backend/app/work_queue/items/billing_items.py`
     alongside charges, while VAT and annual reports live in `tax_items.py`. An advance payment is a
     tax obligation, not a billing artifact.

2. **Generation has three entry points and no rollover.**
   - VAT + advance payments: `backend/app/clients/services/client_onboarding_service.py:92-170`
     (`_sync_vat_work_items`, `_sync_advance_payments`), driven by `obligation_plan`.
   - Annual reports: `backend/app/actions/services/obligation_orchestrator.py`, which does **not**
     use `obligation_plan`.
   - Advance payments additionally have `POST /clients/{id}/advance-payments/generate` and the
     office-wide `POST /advance-payments/bulk-generate`.
   - **VAT has no bulk or office-wide generation at all.**
   - There is no scheduler. `backend/app/lifespan.py` runs signature-expiry and signature
     reconciliation only. Next year's obligations therefore exist only if someone onboards or edits
     a client, or manually runs advance bulk-generate.
   - `client_onboarding_service.py:100` imports the **private** `_years_to_generate` from
     `app.actions.services.obligation_orchestrator`, inside a function body — a layering inversion
     hidden from static import analysis.

3. **Three different coupling styles, one of them a cycle.**
   - VAT → advance payments: **pull**, explicit advisor action (refresh-turnover), frozen snapshot,
     one resolver (`TurnoverLookupRepository._resolve`) with a documented SQL twin. This is the
     model the other couplings should follow.
   - VAT → annual reports: **pull**, explicit advisor action (auto-populate) — but auto-populate
     does **not** invalidate persisted `tax_due` / `refund_due`, while manual income/expense
     mutations do. Readiness then reads a stale persisted value.
   - Advance payments → annual reports: **push**. `advance_payment_service.py:243` calls
     `AnnualReportTaxService.invalidate_tax_if_open` through a deferred function-body import,
     because `annual_reports` already imports `advance_payments` repositories. A real circular
     dependency, worked around rather than resolved.

4. **VAT balance is summed into an income-tax total.**
   `annual_report_tax_service.py:127` computes
   `total_liability = tax_after_credits + ni.total + (vat_balance or 0) - advances_paid`.
   `annual_report_pdf_builder.py:265` renders that number labelled `סה"כ חבות (מס + ביטוח לאומי)` —
   the label omits the VAT term the arithmetic includes. Already recorded as a known issue in
   `docs/domains/annual-reports.md`; still open.

### 3.4 Documentation drift found during this review

The domain docs are `Source of truth: mandatory`, so these are defects in their own right:

- `docs/domains/vat.md` lists `VAT.CLIENT_CLOSED` and `VAT.CLIENT_FROZEN` in the error table. Both
  are dead — `vat_intake_service.py:60-62` records that the VAT-local re-check was unreachable and
  removed once `VatClientContextService` adopted the shared guard.
- `docs/domains/annual-reports.md` known-issue list states that
  `AdvancePaymentService.bulk_mark_paid` does not invalidate persisted tax. It does, at
  `advance_payment_service.py:508`. The VAT-auto-populate half of that same known issue **is** still
  real (§3.3.3).

## 4. Target model

**One obligation, three specializations.** An obligation is
`(client_record, obligation_type, period | tax_year)` and is already anchored physically by
`TaxCalendarEntry` plus the FK each root carries.

Every obligation must answer these five questions the same way. The plan is organized around
closing the gaps in this table.

| # | Question | Canonical answer | Today |
|---|---|---|---|
| 1 | Is it owed? | `obligation_plan` | VAT + advance only; annual excluded |
| 2 | When is it due? | `due_date_effective` | annual uses `filing_deadline` |
| 3 | Is it resolved? | `is_*_resolved` **plus a SQL twin** | Python side only; VAT SQL forked |
| 4 | Is it late? | one rule over #2 and #3 | three different rules |
| 5 | May I act on this client? | `assert_client_record_is_active` | done |

## 4.1 The unified lifecycle

Established with the product owner on 2026-07-27. This section is the target the three domains
converge on. §7 was re-cut against it on 2026-07-27; §9.1 records what that changed.

### 4.1.1 One status set

VAT and annual reports already run the same sequence under different names — nobody decided they
should differ; they were written separately and never compared:

| Stage | VAT today | Annual today |
|---|---|---|
| waiting for input | `pending_materials` | `collecting_docs` |
| input received | `material_received` | — |
| working | `data_entry_in_progress` | `in_preparation` |
| ready, awaiting verification | `ready_for_review` | `pending_client` |
| done | `filed` | `submitted` |
| — | — | `closed` |
| cancelled | `canceled` | `canceled` |

The target is **one status set of six, used in full by all three domains**:

1. **Waiting for input** — the obligation exists; its inputs have not arrived
2. **Input received** — work can start
3. **In progress**
4. **Ready — awaiting verification**
5. **Closed (locked)**
6. **Cancelled**

An obligation that exists is by definition waiting for its inputs, so a separate "not started" state
is redundant and is dropped. For an advance payment the "input" is the filed VAT return; for VAT it
is the client's documents; for an annual report it is the year's material.

**Canonical identifiers** (closes O-8). The enum is `ObligationStatus`, and it lives in
`app/common/enums.py` beside `ObligationType` — the pair states its own relationship:

| # | Value | Label |
|---|---|---|
| 1 | `awaiting_input` | ממתין לחומר |
| 2 | `input_received` | החומר התקבל |
| 3 | `in_progress` | בעבודה |
| 4 | `awaiting_verification` | ממתין לאימות |
| 5 | `submitted` | הוגש |
| 6 | `canceled` | בוטל |

`canceled` keeps the US spelling already used by all three domains. `submitted` is preferred over
`filed`: in English "filed" is bound to tax returns, while the closing act here is the same in all
three — the obligation is **reported and settled**. An advance payment is reported to the authority
as well as paid, which is why the product owner defines its close as "confirmed reported and paid".
`closed` was rejected outright: `AnnualReportStatus.CLOSED` and `ClientStatus.CLOSED` already exist
with two other meanings.

**Migration mapping.** What the mapping exposes is more useful than the mapping itself:

| Domain | Mapping | What it means |
|---|---|---|
| VAT | `pending_materials`→1 · `material_received`→2 · `data_entry_in_progress`→3 · `ready_for_review`→4 · `filed`→5 · `canceled`→6 | a pure rename — every stage already exists |
| Annual | `not_started` + `collecting_docs`→1 · `in_preparation`→3 · `pending_client`→4 · `submitted` + `closed`→5 · `canceled`→6 | two merges, and **stage 2 is empty** — no existing status lands there. It is a new stage, not a rename |
| Advance | `pending`→1 **or** 2 depending on whether the turnover is known · `partial`→3 · `paid`→5 | not a mapping but a **derivation**; stage 4 and `canceled` are both new |

This is §4.1.3's claim restated as counts, and it is now estimable: VAT renames, annual reports gain
one stage, advance payments gain two and lose their money-derived status entirely.

### 4.1.2 Events advance; only a person locks

The precedent already exists in VAT: entering the first invoice moves the item from
`material_received` to `data_entry_in_progress` automatically. Generalised:

> **A data event advances an obligation through the working stages. Only an explicit human action
> locks it.**

| Event | Transition | Driver |
|---|---|---|
| turnover becomes known (advance) | waiting → input received | automatic |
| first invoice entered (VAT) | input received → in progress | automatic |
| payment recorded (advance) | input received → in progress | automatic |
| **paid in full (advance)** | **in progress → awaiting verification** | **automatic** |
| advisor files / submits / confirms | awaiting verification → closed | **human** |

Paying in full therefore *does* move the status — it just does not lock. Before the lock, a change to
the expected amount legitimately moves the record back to "in progress", because there genuinely is
more to do. After the lock, nothing moves except through an amendment.

### 4.1.3 Consequences per domain

- **VAT** — essentially unchanged; its sequence is already correct. Renames only.
- **Annual reports** — loses a whole layer: the `pending_client` stage, the signature request created
  on entering it, the `client_approved_at` readiness gate, and the post-submission `closed` status.
  Readiness drops from four gates to three.
- **Advance payments** — changes most. Gains three stages it does not have, an explicit lock action,
  and an amendment mechanism. Status stops being derived from money, and `partial` stops being a
  status at all: a part-paid advance is simply "in progress" with an outstanding balance. That is
  what it always was — a fact about the amount, not a stage of the lifecycle.

### 4.1.4 Turnover has one source, chosen by the client

The turnover behind an advance payment is not an advisor's choice; it follows from the client:

- **Client reports VAT** → the turnover comes from the VAT return, always. No manual entry, no
  refresh command, no provenance question.
- **Client does not report VAT** (`osek_patur`, exempt) → the turnover is entered manually. These
  clients have no VAT returns at all but do receive advance payments, so this is the only path that
  can supply a figure for them.

**The snapshot moves to the lock, and stops being a command.** The frozen figure exists to stop a
settled period from changing underneath the advisor — but the lock already freezes the whole record,
so freezing the number separately is redundant. Before the lock the figure is live; at the lock it is
written onto the record permanently.

Period mapping survives unchanged: VAT frequency and advance frequency are independent, so one
advance period may span two VAT returns. That resolver is sound — it simply stops writing a snapshot.

### 4.1.5 The transition graph

```
  ┌──────────────────────────────────────────────────┐
  │                                                  ▼
waiting_input ⇄ input_received ⇄ in_progress ⇄ awaiting_verification ──► closed
      │               │               │                   │              (locked)
      └───────────────┴───────────────┴───────────────────┴──► canceled       │
                                                                              │
                                                    amendment ────────────────┘
                                                  (a new record)
```

- **Forward** — one stage at a time. Stages 1→4 may be driven by events (§4.1.2); the move into
  `closed` is always an explicit human action.
- **Backward** — one stage at a time, and only while unlocked. Jumping back several stages is not
  allowed: it destroys the readable history of what actually happened.
- **Cancel** — from any unlocked stage. Not from `closed`: a closed period is a record of a filing.
- **`closed` has no outgoing transition.** Correcting a closed obligation creates a new record
  (§4.1.6); the closed one stays closed permanently.

### 4.1.6 Amendment is a new record, not a reopen

The two existing mechanisms are opposites, and the new-record model wins:

- **VAT** — an amendment is a separate row carrying `is_amendment` and `amends_item_id`, so the
  original stays locked and the chain is visible.
- **Annual reports** — `amend_report` reopens the same row back to `in_preparation`. Under this
  model, "closed" would only ever mean "closed for now". This mechanism retires.

**Finding: VAT's amendment is currently unreachable.** The fields, the validation, and the
cycle detection all exist, but `is_amendment` is set *at filing time* on a row that already exists
(`vat_routes_filing.py:45` → `vat_filing_service.py:70`) — and a second row for the same period can
never be created: `create_work_item` rejects a duplicate active period, and a filed row cannot be
soft-deleted (`VAT.FILED_IMMUTABLE`). So this decision is real work in all three domains, not a copy
of a working VAT feature.

What the model requires:

| # | Requirement | State today |
|---|---|---|
| 1 | The active unique index becomes "one **original** per client+period/year" — amendments excluded | blocks a second row outright |
| 2 | An explicit "create amendment" action from a closed record | does not exist |
| 3 | Chain link + cycle detection | **already implemented** (VAT) |
| 4 | Every aggregate reads only the latest record in the chain | **not handled — would double-count** |

Requirement 4 is the real risk. `sum_net_vat_by_client_record_year`, the annual report's VAT import,
and the advance payments' turnover lookup would all sum the original *and* its amendment. It reaches
outside VAT, because advances draw their turnover from the same figures.

**A chain is one row, everywhere.** Lists, counts, and sums show the latest record in the chain,
marked as amended; the earlier records are reachable from the client card. This is both the product
answer and what keeps requirement 4 from having to be re-solved at every call site.

How a new amendment record is born is settled in §4.1.12 (D-21), and the resulting uniqueness
predicate — which must exclude deleted, amendment, and cancelled rows together — is stated once in
§4.1.13.

### 4.1.7 What "closed" means

**Nothing on a closed record can change.** Not the figures, not the metadata, not the assignee, not
the notes. Every change goes through an amendment. VAT already behaves this way; annual reports and
advance payments do not.

The closing act records the same four facts in all three domains — and three of the four already
exist everywhere, under different names:

| | when | how | reference | by whom |
|---|---|---|---|---|
| VAT | `filed_at` | `submission_method` | `submission_reference` | `filed_by` |
| Annual | `submitted_at` | `submission_method` | `ita_reference` | **missing** |
| Advance | `paid_at` | `payment_method` | `payment_reference` | **missing** |

Only "by whom" is missing, and only in two of the three. The closing act is the one moment an
obligation becomes a record rather than a task; it must always name its author.

**An amendment record carries no due date.** A correction is not a new obligation with its own
deadline, so it is excluded from every lateness calculation. Two consequences that must be handled
rather than discovered:

- `due_date_effective` becomes genuinely nullable, and every overdue query must read NULL as
  "not late" rather than comparing against it.
- `work_queue/items/tax_items.py` `_vat_due_date` currently **raises** when `due_date_effective` is
  None. It would fail on the first amendment.

The amendment still links to the same `TaxCalendarEntry` as the record it corrects — the entry is the
regulatory period, which is shared; only the per-record due-date snapshot is absent.

### 4.1.8 What it takes to close

Closing is gated. Every domain answers the same question — *can this be closed, and if not, what is
missing?* — and returns its own list of unmet gates. Annual reports already publish exactly this as
their readiness check; it generalises to all three rather than staying one domain's feature.

**Shared gate, all three:** an assignee must be set. A closed obligation is a record with an author,
and the office needs to see who owned it. VAT enforces this today (`VAT.ASSIGNEE_REQUIRED`); the
other two do not.

**Per-domain gates:**

| Domain | Gates |
|---|---|
| VAT | a final amount exists — either the computed net VAT or an override with a justification |
| Annual | required schedules complete · total income > 0 · a tax result (`tax_due` or `refund_due`) persisted |
| Advance | the turnover is known and the expected amount is computed |

**Payment in full is not a gate on an advance.** The advisor decides when a period is settled: a
client who paid less, finally, is a closed period with an outstanding difference — not an open one
forever. Closing means "this is what was reported and paid", not "the arithmetic balances".

This does not contradict D-8. The automatic move to "awaiting verification" on full payment is a
**shortcut, not the only route**: a human can always advance a stage manually, and an advisor closing
a part-paid period simply moves it forward themselves. Events save keystrokes; they never own the
path.

### 4.1.9 Who does what

There are two roles, `ADVISOR` and `SECRETARY`. The division follows the lifecycle rather than the
domain:

> **A secretary moves an obligation through the working stages. Only an advisor closes it, sends it
> back, cancels it, or amends it.**

VAT already works this way. The other two do not:

| | Secretary today | Under this model |
|---|---|---|
| VAT | opens periods, receives material, enters invoices, moves to review | unchanged |
| Annual | annexes, details, financial lines | unchanged |
| Advance | **read-only — every write is advisor-only** | may record a payment and move stages |

The advance-payments restriction is the outlier and retires. Recording a payment that arrived is
clerical work, not a judgement; the judgement is the close, and that stays with the advisor.

Advisor-only actions, in all three: **close · send back · cancel · amend · delete**.

### 4.1.10 What "input received" means per domain

Stage 1 → 2 is the point where the obligation has what it needs to be worked. Each domain names its
own input, and each may open the gate automatically:

| Domain | Input | Opens |
|---|---|---|
| VAT | the client's documents | an advisor or secretary marks them received |
| Advance | the period's VAT return, closed | automatically, when the return closes |
| Annual | **both** — the tax year's VAT periods are all closed **and** the year's documents are marked received | when both hold |

The annual report needs both because it draws on more than VAT: forms 106, securities statements,
and rental income never pass through a VAT return.

Two notes for implementation:

- For a client who files no VAT (`osek_patur`), "all the year's VAT periods are closed" is vacuously
  true, and the annual gate reduces to documents alone. This mirrors D-9, where the same client's
  turnover falls back to manual entry.
- "Closed" here means the VAT period reached a terminal state — filed **or** cancelled. A cancelled
  period contributes no figures, which is a fact about the year rather than a blocker.

### 4.1.11 Lateness

Urgency is already shared: `work_queue/items/common.py` `urgency()` grades every source the same way
(overdue / approaching / important / upcoming). Only the *entry* rule differs per domain, and D-1
fixes that. Nothing here changes the grading.

Two different questions are involved, and they are answered differently:

- **"Is it late right now?"** — computed, never stored: the obligation is open and today is past
  `due_date_effective`. This is what colours a row and places it in the queue.
- **"Was it closed late?"** — **recorded on the record at the moment of closing.** A historical fact
  about how the office performed, and it must not depend on a later reading of a due date that may
  itself have moved.

Recording it at the close is deliberate: a real-time "went overdue on date X" marker would be
strictly better history — it survives a later extension — but it needs a daily sweep, and there is no
scheduler in this system (`lifespan.py` runs startup tasks only). The closing act needs no
infrastructure at all. Revisit only if a scheduler arrives for **O-1**.

Advance payments already compute `paid_late` this way at read time; it becomes a stored fact written
once, and the same field appears on all three.

### 4.1.12 Cancelling, deleting, and the birth of an amendment

**An amendment is born as a full copy of its original** — every invoice, line, and figure — and opens
at "in progress". The material already exists; only the figures are wrong. It never starts at
"waiting for input", because nothing is being waited for. (Closes O-5.)

**An obligation is never removed on its own — it is replaced.** A regulatory duty does not stop
existing because nobody has started working on it. Removing one is legitimate in exactly two
situations, and both are consequences of correcting the client's configuration rather than actions
against the obligation itself:

- **Replacement** — the duty exists in a different shape. A client wrongly configured as monthly owes
  six bi-monthly VAT periods, not twelve monthly ones. Each removal is paired with a creation.
- **Never liable** — the client was never subject to this obligation at all (wrong entity type).
  Nothing replaces it, because nothing was owed.

**The direct `DELETE` endpoints therefore retire in all three domains.** Obligations disappear only
through reconciliation: the advisor fixes the client's configuration, and the system removes what is
no longer owed and creates what now is. This model already exists and works — advance payments'
`cleanup_stale_cadence` does exactly this on a frequency change. Nobody "deletes an advance" there;
they change the client's cadence and the schedule reconciles.

**Why a direct delete is more dangerous than it looks: the system can only show obligations that
exist, never ones that are missing.** The overdue list shows reports that exist and are past due; a
deleted report is not late, it is simply absent. Nothing anywhere says "this client has no annual
report for 2026". Deleting an obligation removes the duty *and* the ability to discover that it is
gone. Recreation is not automatic either — `generate_client_obligations` runs only on client creation
and on a client update that touches an obligation field, and there is no rollover (O-1). A deleted
open report returns only if someone happens to edit that client.

**Residual risk, accepted:** blocking deletion closes this hole but not the whole family. An
obligation can also be missing because a year was never opened (O-1) or because a client never had a
frequency configured. A partial safeguard already exists — the advance-payments
`bulk-generate/preview` endpoint reports active clients with no configured frequency — but there is
no general "missing obligations" view, and by D-25 none is planned.

**Cancelling and deleting are different facts, and both are needed:**

| | Cancelled | Deleted (soft) |
|---|---|---|
| Was the obligation ever real? | **yes** | **no** |
| What happened | it was real and stopped being the office's to handle | the system created it from a wrong configuration |
| Visible | yes, as a lifecycle state | no — hidden, audit trail preserved |

The mechanical reason deletion cannot simply be replaced by cancellation: the active unique index
ignores deleted rows, so **soft delete is what frees the period slot** for the correct obligation to
be created. A client wrongly configured as monthly gets twelve VAT periods; six of them are not
obligations in law, and the correct bi-monthly periods cannot be created while the wrong rows still
occupy their slots. Marking them "cancelled" would leave the slots taken.

**Cancelling is terminal, and a cancelled row releases its slot.** A closed client who returns is a
re-onboarding: the period is created fresh, and the cancelled row stays visible as history — "we had
this, it was cancelled when the client left". Reviving the old row to the stage it stopped at would
pretend the gap never happened; its material is stale, and someone else may have filed in the
meantime. This is the same reasoning as D-10: a correction is a new record, not a reopen.

A cancellation made in error is not undone by reviving — cancelling is an advisor-only action
(D-17), and the fix is to create the period again.

### 4.1.13 The uniqueness rule, in one place

Three separate decisions now narrow what "one obligation per period" means. Stated together, because
it is a compound rule and each part was decided for a different reason:

> Within one client and one period (or tax year), there is at most one row that is
> **not deleted**, **not an amendment**, and **not cancelled**.

| Exclusion | Why | Decision |
|---|---|---|
| deleted | frees the slot when the obligation was never real | D-22 |
| amendment | a correction is a second row for the same period, by design | D-10 |
| cancelled | a returning client must be able to have the period created fresh | D-23 |

Every one of these is a change to the existing partial unique index in all three domains. Getting the
predicate wrong reintroduces exactly the conflicts each exclusion was added to prevent.

**A closed obligation can never be deleted.** It is a record of a filing or a payment, not a task.
This is already enforced in VAT (`VAT.FILED_IMMUTABLE`) — and **in neither of the other two**:

- `annual_report_service.py:60` `delete_report` checks only that the report exists. **A submitted
  annual report can be soft-deleted.**
- `advance_payment_service.py:592` `delete_payment_for_client` checks only ownership. **A paid
  advance can be soft-deleted.**

Both must adopt the VAT guard. Verified at the route layer too — neither has a status check there
either. Each of the three protects something different, and no two overlap:

| | who may delete | blocked by status | reason required |
|---|---|---|---|
| VAT | advisor **and secretary** | ✅ filed blocked | ❌ |
| Annual | advisor | ❌ none | ❌ |
| Advance | advisor | ❌ none | ✅ |

VAT is the only one that guards the status and the only one that lets a secretary delete — which also
contradicts D-17.

**A paid advance's deletion leaves the annual report holding a stale tax figure.** Deleting removes
the row from `sum_paid_by_client_year`, but `_soft_delete_payment`
(`advance_payment_service.py:610`) does **not** call `_invalidate_annual_report_tax`. Every other
write path does, or documents why it does not:

| Path | Invalidates | |
|---|---|---|
| create / update | ✅ | |
| `bulk_mark_paid` | ✅ | |
| `bulk_update_rate_from_period` | ❌ | deliberate, with a comment — it only touches PENDING rows |
| **delete** | ❌ | **no comment, no reason** |

The likely cause: `_soft_delete_payment` is shared between the advisor's delete and the stale-cadence
cleanup. The cleanup is PENDING-only, and a PENDING row has `paid_amount == 0`, so it can never move
a row into or out of the PAID set — no invalidation needed there. The advisor's delete inherited the
helper *and its assumption*, but places no status restriction at all. Same family as the
`bulk_mark_paid` defect that was already found and fixed; this path was missed.

D-22's guard closes both holes at once: if a closed obligation cannot be deleted, a paid advance
cannot be deleted, and the invalidation question never arises.

**Client reminders stay blocked once the deadline has passed.** This is current VAT behaviour
(`notification_policy_service.py`, `מועד הגשת מע"מ כבר חלף`) and it is intentional: after the date,
chasing the client is a conversation, not an automated reminder. Recorded here so it is not "fixed"
later as an oversight.

## 4.2 Reconciliation — the only way an obligation is removed

D-24 says an obligation is never removed on its own, only replaced. This section specifies the
mechanism that does the replacing.

### 4.2.1 What it is

Reconciliation compares **what the client owes** against **what rows exist**, and closes the gap. It
is never invoked against an obligation; it is invoked against a client whose configuration changed.

The working precedent is `AdvancePaymentService.generate_annual_schedule`'s stale-cadence handling.
Its design is sound and is lifted wholesale rather than redesigned — the rules in §4.2.3 are its
rules, generalised.

### 4.2.2 What triggers it

A change to a client field that determines what is owed. The set already exists as
`CLIENT_OBLIGATION_TRIGGER_FIELDS` (`client_constants.py:46`):

| Field | Affects |
|---|---|
| `entity_type` | VAT liability, advance liability, annual report form/type |
| `vat_reporting_frequency` | which VAT periods are owed |
| `advance_payment_frequency` | which advance periods are owed |

**The trigger set is right; the handler is not.** `client_update_service.py:92` fires
`generate_client_obligations` on any of these — and that function creates **annual reports only**
(`ObligationResult` carries a single field, `reports_created`). Changing a client's VAT frequency
therefore triggers annual-report generation and touches no VAT period at all. Advance payments'
stale-cadence cleanup exists but runs only when someone explicitly calls `generate`, never as a
consequence of the frequency change itself.

So today: one trigger set, three obligation types, and only one of them reconciles — the wrong one.

### 4.2.3 How existing rows are classified

For each obligation type, reconciliation computes what the client owes under the **new**
configuration and classifies every existing row against it:

| Row | Action | Why |
|---|---|---|
| matches the new plan | keep | already correct |
| superseded · not started · due date ahead | **removable** | nothing was done, nothing is owed |
| superseded · **past due** | **keep** | an unpaid period whose date has passed is a real debt, not a leftover |
| superseded · worked or closed | **keep and report** | that part of the year stays on the old shape until a human resolves it |

The past-due rule is the one most easily lost in a rewrite, and it is load-bearing: the difference
between a stale row and a debt is the calendar, not the configuration.

### 4.2.4 Safety rules

These are the properties that make reconciliation safe to run automatically. All four come from the
existing implementation:

1. **All or nothing per year.** If removable rows exist and cleanup is not confirmed, create
   **nothing**. Generating only the periods the stale rows do not occupy is exactly how a month ends
   up covered by two cadences at once.
2. **Confirmation is required before removal.** The first call reports the counts and creates
   nothing; the caller repeats with the flag. Removal is never a silent side effect of an edit.
3. **What is settled is never removed.** It is reported instead, split from the removable rows,
   because the two demand different follow-ups.
4. **Removal is resolved before the creation loop, not inside it.** The existence check matches on
   the period key alone, so a superseded row makes the generator skip the very period it should
   replace.

Every removal is audited with a system-written reason (`advance_payment.deleted` carries one today),
because no human typed one.

### 4.2.5 What it must become

| Domain | Reconciliation today | Needed |
|---|---|---|
| Advance | exists, but only inside an explicit `generate` call | run it on the configuration change itself |
| VAT | **none** — `_sync_vat_work_items` only creates, and only at client creation | full reconciliation |
| Annual | **none** — the orchestrator only creates | full reconciliation |

One service reconciles all three (P8's `ObligationGenerationService`), driven by one trigger set,
returning one report of what was created, what was removed, and what could not be touched.

### 4.2.6 Two shapes, because the obligations are not shaped alike

§4.2.3 describes replacing a **set** of periods. That fits VAT and advance payments, which have many
periods per year. An annual report is **one record per year**, so a change does not replace a set —
it invalidates a single record.

**Shape A — set replacement (VAT, advance payments).** A frequency change replaces which periods are
owed. Classify each row per §4.2.3. A period already **in progress** is kept and reported, never
removed: its invoices belong to *months*, and the months did not move — only the grouping did. The
work is still valid.

**Shape B — record replacement (annual reports).** An `entity_type` change alters the primary form
(`1301` / `1214` / `1215`), and with it the tax engine, the required schedules, and the meaning of
the income and expense lines. The existing record does not become mis-grouped; it becomes **wrong**.
It is therefore removed and recreated with the correct form.

| Report state | Action |
|---|---|
| waiting for input · input received · in progress | removed and recreated |
| **closed** | **never removed** — kept and reported (D-13, D-22) |

**Removing a worked annual report destroys its income and expense lines**, and unlike Shape A there
is nothing to preserve them in. This must be stated in the confirmation prompt, not discovered
afterwards. The confirmation requirement of §4.2.4 rule 2 applies here with more force, not less.

The asymmetry between the shapes is deliberate and worth stating plainly: a frequency change
re-groups valid work, while a form change invalidates it.

## 4.3 Edge cases found reviewing §4.1 and §4.2

A full read-back of the specification on 2026-07-27. Ten items: two internal contradictions, five
undefined cases, three decisions undermined elsewhere. None invalidated the model. **All ten are now
resolved** — each item below carries its resolution, and the resulting decisions are D-29 … D-38.

A pattern worth recording: not one of these came from a wrong decision. Every one was produced by
**correct decisions taken at different times intersecting** — which is the argument both for having
done this pass and for repeating it once the remaining open questions close.

### Contradictions — the specification disagrees with itself

**EC-1 · The stated reason for keeping soft delete is no longer true.**
§4.1.12 justifies soft delete mechanically: *"marking them cancelled would leave the slots taken"*.
But D-23 later excluded cancelled rows from the uniqueness rule (§4.1.13) — so cancelling **does**
free the slot. The justification was written before D-23 and never revisited. The distinction between
cancel and delete may still be worth keeping, but only on semantic grounds ("was the obligation ever
real?"); the mechanical argument must be struck.
→ **Resolved (D-29):** the mechanical claim is struck. The semantic distinction stands on its own.

**EC-2 · An amendment erases the record that the original was late.**
D-12 shows an amendment chain as one row, the latest. D-14 gives an amendment no due date and excludes
it from lateness. D-20 records "closed late" at the close. A period filed two months late and then
amended therefore presents as a row with no due date and no lateness — **amending a period hides that
it was late**, which is exactly the fact D-20 exists to preserve.
→ **Resolved (D-34):** the chain presents the **original's** lateness. The period was late once, and
that is a fixed fact; a correction changes figures, not history.

### Undefined cases

**EC-3 · The annual input gate can open on an incomplete year.**
§4.1.10 opens the gate when "all the year's VAT periods are closed". "All" is evaluated over rows that
*exist*, not periods that are *owed* — and with no rollover (O-1) a year may hold eight of twelve
monthly periods. Eight closed rows would open the gate and the annual report would be built on a
short year, silently.
→ **Resolved (D-30):** every gate that asks "are they all closed?" evaluates against the **obligation
plan** — what is owed — never against the rows that happen to exist. This is already the canonical
answer to question #1 in §4; the gate simply was not using it.

**EC-4 · An `osek_patur` advance payment is stuck at both ends.**
Its input gate opens on "the period's VAT return, closed" (§4.1.10) — and this client has no VAT
returns at all. Its closing gate requires "the turnover is known" (§4.1.8), and D-9 says that
turnover is entered manually — but manual entry is **not** listed as an event that opens the input
gate (§4.1.2). So the record cannot leave stage 1 and cannot ever be closed.
→ **Resolved (D-31):** for a client with no VAT reporting, **manual turnover entry is the input
event** — the same role the closed VAT return plays for a VAT filer, from the other source. This is
the symmetric half of D-9; without it D-9 produces a permanently stuck client, which was never the
intent.

**EC-5 · A payment that arrives before the turnover is known.**
§4.1.2 starts the payment events at "input received". The advance-payments documentation states
plainly that recording a payment before its VAT return arrives is normal. What a payment recorded at
stage 1 does is undefined.
→ **Resolved (D-35):** a recorded payment advances the record to "in progress" from wherever it was.
An advance's input is therefore satisfied by **either** signal — the turnover arriving or money
arriving; both are evidence the period is live.
→ **And a consequence that had to be settled with it:** this crosses two stages at once, which
§4.1.5 appears to forbid. It does not. The event **performs both transitions** (1→2, then 2→3) and
both are recorded; it does not jump over stage 2. The "one stage at a time" rule exists to keep
history readable — it constrains skipping a stage's meaning, not how many transitions one event may
cause. Backward movement remains strictly one stage per action (D-11).

**EC-6 · "Closed late" is meaningless for an amendment.**
An amendment has no due date (D-14) but is closed like anything else (D-20). The recorded value must
be explicitly null / not-applicable rather than `false`, or every amendment will read as "closed on
time".
→ **Resolved (D-32):** "closed late" is **null** on any record without a due date. Never `false`.

**EC-7 · Backward transitions may or may not need a reason.**
§4.1.9 lists "send back" as an advisor action. VAT requires a non-empty correction note today. The
specification does not say whether that survives, nor whether it applies to every backward move or
only 4 → 3.
→ **Resolved (D-33):** **every** backward transition requires a reason. This is D-11's own rationale
carried through — backward movement is kept one stage at a time so the history stays readable, and a
readable history without a "why" is half a history. VAT's existing send-back note generalises.

### Decisions undermined elsewhere

**EC-8 · A secretary can remove obligations.**
`PATCH /clients/{client_record_id}` (`client_routes.py:172`) carries **no `require_role`
dependency at all**. Under D-24, editing a client's frequency is the *only* remaining way to remove
obligations — so a secretary changing a VAT frequency removes VAT periods, which D-17 reserves for an
advisor. D-24 created this leak by moving removal onto an unguarded path.
→ **Resolved (D-36):** the obligation trigger fields — `entity_type`, `vat_reporting_frequency`,
`advance_payment_frequency` — become **advisor-only**. The rest of the client record stays editable by
a secretary; a phone number is not an obligation. The guard belongs on the fields, not on the
endpoint.

**EC-9 · D-9 removes the only detector for a VAT figure that moved after the lock.**
The retired turnover-mismatch mechanism was what surfaced "the VAT number has changed since we set
this". After the lock the advance is correctly frozen — but if the VAT return is later amended,
nothing says so. There was a detector; after D-9 there is none.
→ **Resolved (D-37):** a detector returns, narrowed. It compares a **locked** advance's stored
turnover against the current latest record in its VAT chain, and flags the divergence so the advisor
can decide whether to issue an amendment. Nothing else D-9 retired comes back: no `turnover_source`,
no `turnover_snapshot_at`, no refresh commands, no `available_turnover` / `missing_turnover`. The
existing `vat_turnover_mismatch_expr` is the right shape for this and should be narrowed to locked
rows rather than deleted.

**EC-10 · Shape B leaves a dangling `annual_report_id` (dormant).**
`AdvancePayment.annual_report_id` points at a report. §4.2.6 Shape B removes a report and creates a
new one; the advances keep pointing at the removed row. **Dormant, not live:** no application code
populates this column today — only the seed builders — but the create API accepts it.
→ **Resolved (D-38):** reconciliation clears the link when it removes a report. And because no
application code populates the column, it is recorded as a **removal candidate** — a nullable FK that
nothing maintains is a liability, not a feature.

## 5. Decisions taken

| # | Decision | Date |
|---|---|---|
| D-1 | VAT work-queue entry aligns to `due_date_effective` with the same upcoming window as advance payments and annual reports. Accepted consequence: VAT periods become visible in the queue **before** they are late, which is a visible product change. | 2026-07-27 |
| D-2 | The advance-payments → annual-reports dependency is inverted with an **explicit port** — a named interface that `annual_reports` implements and `advance_payments` depends on. An event/hook registry was rejected: it hides ordering and makes "who invalidated this figure" untraceable, against ADR-0003. | 2026-07-27 |
| D-3 | The unified bi-monthly alignment gate raises a **single `TAX_CALENDAR` error code**. `VAT.INVALID_PERIOD_FOR_FREQUENCY` and `ADVANCE_PAYMENT.INVALID_PERIOD` are retired for this condition. Accepted consequence: an API contract change — `openapi.json` + `generated.ts` regenerate, and both frontend message catalogs need updating. | 2026-07-27 |
| D-4 | **One lifecycle, one status set of six** (§4.1.1), used in full by all three domains. All three are the same kind of thing: an obligation completed by a deadline, locked on completion, changeable afterwards only by amendment. "Filed", "submitted", and "paid" are three names for the same closing act. | 2026-07-27 |
| D-5 | **No client signature or client review on an annual report.** The office reviews and files alone. The `pending_client` stage, the signature request created on entering it, and the `client_approved_at` readiness gate all retire; readiness drops from four gates to three. The `signature_requests` module itself stays — it also serves engagement agreements, powers of attorney, and custom documents — but the `ANNUAL_REPORT_APPROVAL` type disconnects. `VAT_RETURN_APPROVAL` is defined and referenced by nothing; it retires with it. | 2026-07-27 |
| D-6 | Annual reports' post-submission `closed` status **adds nothing** and is deleted. Closing is the same single act in all three domains. | 2026-07-27 |
| D-7 | **Advance payments gain an explicit lock and an amendment mechanism.** Locking is the advisor confirming the period was reported and paid — never an arithmetic consequence. `partial` therefore stops being a status: a part-paid advance is "in progress" with an outstanding balance. This closes the existing defect where a turnover refresh could silently drop a settled payment back to PARTIAL. | 2026-07-27 |
| D-8 | **Events advance an obligation through the working stages; only a person locks it** (§4.1.2). Paying in full moves an advance to "awaiting verification" — it does not lock it. | 2026-07-27 |
| D-10 | **Amendment is a new record, not a reopen** (§4.1.6). The closed record stays closed permanently and the chain is visible. Annual reports' `amend_report` reopen mechanism retires. Note that VAT's version of this is currently unreachable, so this is real work in all three domains. | 2026-07-27 |
| D-11 | **Backward transitions move one stage at a time, and only while unlocked.** Multi-stage jumps are rejected because they destroy the readable history of what happened. | 2026-07-27 |
| D-12 | **An amendment chain is one row everywhere** — lists, counts, and sums use the latest record in the chain, marked as amended; earlier records are reachable from the client card. This is what stops every aggregate from having to solve double-counting on its own. | 2026-07-27 |
| D-13 | **Closing is a full lock** (§4.1.7): nothing on a closed record changes — figures, metadata, assignee, or notes. Every change goes through an amendment. VAT already behaves this way; annual reports and advance payments do not. The closing act must also record **who** closed it, which today only VAT does. | 2026-07-27 |
| D-14 | **An amendment record has no due date** and is excluded from every lateness calculation — a correction is not a new obligation. Requires `due_date_effective` to be treated as genuinely nullable across all overdue logic. | 2026-07-27 |
| D-15 | **An assignee is required before closing, in all three domains** (§4.1.8). VAT enforces this today; annual reports and advance payments do not. Closing gates generalise into one shared shape — "can this be closed, and what is missing" — which annual reports already publish as their readiness check. | 2026-07-27 |
| D-16 | **Payment in full is not a gate on closing an advance.** The advisor decides when a period is settled; a client who finally paid less closes with an outstanding difference. This does not weaken D-8 — the automatic move on full payment is a shortcut, and a human may always advance a stage manually. | 2026-07-27 |
| D-17 | **A secretary moves an obligation through the working stages; only an advisor closes, sends back, cancels, amends, or deletes** (§4.1.9). VAT already works this way. Advance payments' blanket advisor-only write restriction retires — recording a payment that arrived is clerical work, not a judgement. | 2026-07-27 |
| D-18 | **An annual report's input gate needs both** the tax year's VAT periods all closed **and** the year's documents marked received (§4.1.10). It draws on material that never passes through a VAT return — forms 106, securities, rental income. For a client who files no VAT the VAT half is vacuously satisfied. | 2026-07-27 |
| D-19 | **Client reminders stay blocked once the deadline has passed** (§4.1.11). Current VAT behaviour, and intentional — after the date, chasing the client is a conversation, not an automated reminder. Recorded so it is not "fixed" later as an oversight. | 2026-07-27 |
| D-20 | **"Closed late" is recorded on the record at the moment of closing**; "late right now" stays computed (§4.1.11). A real-time overdue marker would be better history but needs a daily sweep, and no scheduler exists. Revisit only if **O-1** brings one. | 2026-07-27 |
| D-21 | **An amendment is born as a full copy of its original and opens at "in progress"** (§4.1.12). The material already exists; only the figures are wrong. Closes O-5. | 2026-07-27 |
| D-22 | **Cancelling and deleting stay distinct** (§4.1.12): cancelled = the obligation was real and stopped being the office's; deleted = it was never real, created from a wrong configuration. Soft delete also serves a mechanical purpose — it frees the period slot in the active unique index so the correct obligation can be created. **A closed obligation can never be deleted**, which today only VAT enforces: a submitted annual report and a paid advance can both be soft-deleted. | 2026-07-27 |
| D-23 | **Cancelling is terminal, and a cancelled row is excluded from the uniqueness rule** (§4.1.12). A returning client gets the period created fresh; the cancelled row stays visible as history. Reviving the old row would pretend the gap never happened. Closes O-6. Consolidated with D-10 and D-22 into one uniqueness predicate in §4.1.13. | 2026-07-27 |
| D-24 | **An obligation is never removed on its own — it is replaced** (§4.1.12). A regulatory duty does not stop existing because nobody started it. The direct `DELETE` endpoints retire in all three domains; obligations disappear only through reconciliation after the client's configuration is corrected. Advance payments' `cleanup_stale_cadence` is the working model to generalise. | 2026-07-27 |
| D-26 | **Reconciliation is specified in §4.2 and generalises the existing advance-payments stale-cadence design** rather than inventing one: past-due rows are debts and are never removed; settled rows are reported, not removed; removal requires confirmation and is all-or-nothing per year; removal is resolved before the creation loop. It runs on the client-configuration change itself, not only inside an explicit generate call. | 2026-07-27 |
| D-27 | **Reconciliation has two shapes** (§4.2.6). VAT and advance payments replace a *set* of periods; an in-progress period is kept and reported, because its invoices belong to months and only the grouping changed. Annual reports replace a *record*: an `entity_type` change alters the primary form, the tax engine, and the meaning of every line, so an open report is removed and recreated. A **closed** report is never removed. | 2026-07-27 |
| D-28 | **Removing a worked annual report destroys its income and expense lines**, and nothing preserves them. The confirmation prompt must say so explicitly — this is the one reconciliation outcome that loses work, and it must never happen silently. | 2026-07-27 |
| D-25 | **No "missing obligations" view.** Preventing removal is deemed sufficient. Accepted residual risk: an obligation can still be missing because a year was never opened (O-1) or a client never had a frequency configured — the system can only display obligations that exist, never ones that should. | 2026-07-27 |
| D-29 | EC-1 — the **mechanical** justification for soft delete is struck: D-23 frees the slot on cancellation too. Only the semantic distinction ("was the obligation ever real?") survives. | 2026-07-27 |
| D-30 | EC-3 — every "are they all closed?" gate evaluates against the **obligation plan**, never against the rows that happen to exist. Otherwise a short year opens the annual gate silently. | 2026-07-27 |
| D-31 | EC-4 — for a client with no VAT reporting, **manual turnover entry is the input event**. The symmetric half of D-9; without it that client's advance is stuck at both ends. | 2026-07-27 |
| D-32 | EC-6 — "closed late" is **null** on a record with no due date, never `false`. | 2026-07-27 |
| D-33 | EC-7 — **every** backward transition requires a reason, generalising VAT's send-back note. D-11's rationale carried through: a readable history without a "why" is half a history. | 2026-07-27 |
| D-34 | EC-2 — an amendment chain presents the **original's** lateness. A correction changes figures, not history. | 2026-07-27 |
| D-35 | EC-5 — a recorded payment advances an advance to "in progress" from wherever it was; its input is satisfied by **either** the turnover or the money. The event performs both transitions and records both — it does not jump a stage. "One stage at a time" constrains skipping a stage's meaning, not how many transitions one event causes; **backward** movement stays strictly one stage per action. | 2026-07-27 |
| D-36 | EC-8 — the obligation trigger fields (`entity_type`, `vat_reporting_frequency`, `advance_payment_frequency`) become **advisor-only**; the rest of the client record stays editable by a secretary. The guard belongs on the fields, not the endpoint. Closes the leak D-24 opened. | 2026-07-27 |
| D-37 | EC-9 — a narrowed detector returns: a **locked** advance whose stored turnover diverges from the current latest record in its VAT chain is flagged, so the advisor can decide whether to amend. Nothing else D-9 retired comes back. `vat_turnover_mismatch_expr` is narrowed to locked rows rather than deleted. | 2026-07-27 |
| D-38 | EC-10 — reconciliation clears `AdvancePayment.annual_report_id` when it removes a report, and the column is recorded as a **removal candidate**: no application code populates it. | 2026-07-27 |
| D-39 | **The six stages are `ObligationStatus` in `app/common/enums.py`** (§4.1.1), beside `ObligationType`: `awaiting_input` · `input_received` · `in_progress` · `awaiting_verification` · `submitted` · `canceled`. `submitted` over `filed` — the closing act is the same in all three, an obligation *reported and settled*, and an advance is reported to the authority as well as paid. `closed` rejected: two other enums already use it. Closes O-8. | 2026-07-27 |
| D-40 | **`ReportStage` and `POST /annual-reports/{id}/transition` retire.** Not a product judgement — the layer is dead and lossy. `STAGE_TO_STATUS` (`annual_report_constants.py:50`) maps `in_progress` and `final_review` to the *same* status, omits `post_submission` entirely (so that value fails), and `client_signature` dies with D-5. The two remaining stages are plain aliases for `collecting_docs` and `submitted`. In the frontend the enum and endpoint appear **only in `generated.ts`** — no `endpoints.ts` entry, no call site. A lossy alias over seven statuses has nothing left to abstract once there are six shared ones. Closes O-9. | 2026-07-27 |
| D-41 | **The shared contract gets its own mandatory doc**: a new `docs/domains/tax-lifecycle.md`, `Source of truth: mandatory`, written **after** implementation from the finished code. It is needed because this plan archives when the work lands, and the three domain docs are structurally forbidden from holding it — each one's scope block says *"This file must not contain: other domains' behavior"*, and the shared rules (the six statuses, the transition graph, the reconciliation contract, the uniqueness predicate) belong to no single domain. Splitting them across three would recreate exactly the drift that started this: VAT and annual reports ran the same sequence under different names because nobody compared. Closes O-2. | 2026-07-27 |
| D-42 | **Both of `ADVANCE_PAYMENT.INVALID_PERIOD`'s conditions move to `TAX_CALENDAR`**, so the code retires entirely. Bi-monthly alignment and unsupported `period_months_count` each have an exact twin in the calendar (`_validate_period_alignment`; `_periodic_rule_type`, whose rule map knows only 1 and 2 — the same set as `SUPPORTED_PERIOD_MONTH_COUNTS`). Splitting them would leave two identical validations answering from two namespaces. Closes O-3 and fixes D-3's scope. | 2026-07-27 |
| D-9 | **Turnover has exactly one source, and the client decides which** (§4.1.4). VAT-reporting clients draw from the VAT return, always; clients with no VAT reporting have it entered manually. The refresh commands (single and bulk), `turnover_source` as a stored choice, `turnover_snapshot_at`, `available_turnover`, `missing_turnover`, and the entire turnover-mismatch mechanism (`vat_turnover_mismatch_expr`, the `vat_mismatch` filter, `MonthBatchSummary.vat_mismatch_count`) all retire — a mismatch is impossible when there is one source. The figure freezes at the lock, not by a command. | 2026-07-27 |

## 6. Decisions still open

| # | Question | Why it blocks | Blocks |
|---|---|---|---|
| O-1 | **Rollover policy.** Should next year's obligations be opened by a scheduled job, by an advisor-triggered office-wide generate, or both? Today it is neither reliably — see §3.3.2. | Determines whether P8 needs scheduler infrastructure that does not exist today, and what happens to a client whose frequency is configured mid-year. | P8 |
| ~~O-2~~ | ~~Where does the shared contract live?~~ **Closed 2026-07-27 by D-41:** a new `docs/domains/tax-lifecycle.md`, `Source of truth: mandatory`, written after implementation. | — | closed |
| ~~O-3~~ | ~~Does the "unsupported `period_months_count`" condition also move to `TAX_CALENDAR`?~~ **Closed 2026-07-27 by D-42:** both conditions move; `ADVANCE_PAYMENT.INVALID_PERIOD` retires entirely. | — | closed |
| ~~O-5~~ | ~~Which stage does a new amendment record start in?~~ **Closed 2026-07-27 by D-21:** a full copy of the original, opening at `in_progress`. | — | closed |
| ~~O-6~~ | ~~Can a cancelled obligation be revived?~~ **Closed 2026-07-27 by D-23:** terminal, and excluded from the uniqueness rule so a returning client's period can be created fresh. | — | closed |
| ~~O-8~~ | ~~The six stages have no canonical identifiers.~~ **Closed 2026-07-27 by D-39:** `ObligationStatus` in `app/common/enums.py`, values and migration mapping in §4.1.1. | — | closed |
| ~~O-9~~ | ~~Does `ReportStage` + `POST /{id}/transition` survive?~~ **Closed 2026-07-27 by D-40:** it retires. Not a product judgement — the layer is dead and lossy. | — | closed |
| O-7 | Under D-24, what removes an obligation created for a span the client was not yet liable in — e.g. a VAT period for 2026-01 on a client registered in 2026-06? The frequency is correct, so reconciliation will not touch it, and the direct delete is gone. Either reconciliation must also consider the client's liability start date, or one narrow removal path must survive. | Leaves a class of wrong obligations with no removal route. | P4, P8 |
| ~~O-4~~ | ~~Should advance payments keep a money-derived status or gain an explicit lifecycle?~~ **Closed 2026-07-27 by D-7 and D-8:** explicit lifecycle, explicit lock, `partial` retired as a status. | — | closed |

## 7. Phases

Re-cut 2026-07-27 against §4.1 – §4.3 (D-4 … D-42). The previous cut was written for a consolidation
refactor and is superseded; §9.1 recorded why it had to be redone.

### 7.1 What the specification did to the size of the work

The original plan was five cleanup phases and one structural phase. The specification turned it into
a lifecycle replacement — and §4.1.1's migration mapping sizes each domain honestly, which the
earlier estimate could not:

| Domain | Scale | Why |
|---|---|---|
| **VAT** | a rename, plus two bug fixes | every stage it needs already exists; the mapping is 1:1 |
| **Annual** | loses a layer, gains a stage | the signature flow, `pending_client`, post-submit `closed`, and `ReportStage` all retire; stage 2 is new |
| **Advance** | deepest | two new stages, an explicit lock, an amendment mechanism, and its status stops being arithmetic |

The work is therefore **not** evenly spread, and the domain that looked simplest at the start —
advance payments, three statuses and no graph — is the one that changes most.

### 7.2 Ordering

Two constraints drive it. **§0**: structural work is cheap only while there is no production data.
**Dependency**: everything reads the status enum, so it lands first.

| # | Phase | Gates on |
|---|---|---|
| **P0** | The status enum and the transition graph | — |
| **P1** | Delete provable duplication | — (runs alongside P0) |
| **P2** | Closing and locking | P0 |
| **P3** | Amendment and the uniqueness rule | P0, P2 |
| **P4** | Removal and reconciliation | P0, P3 |
| **P5** | The deadline shape | P3 |
| **P6** | Domain surgery | P0 – P5 |
| **P7** | Coupling and arithmetic | P2 |
| **P8** | Generation and rollover | P4 · blocked on **O-1**, **O-7** |
| **P9** | Documentation | all |

P1 shares no files with P0. P7 is independent of P3 – P6 and may land any time after P2.

### 7.3 What no phase owns yet

Three bodies of work are dragged along by P0 and are not written into any phase above. They are not
optional, and two of them are larger than several of the phases.

| Area | Size today | Why it moves |
|---|---|---|
| **Frontend** | 13 files hold the old status literals | status maps, labels, colours, filters, URL params and message catalogs across three features. Plus the retired endpoints (`/transition`, the `DELETE`s, the refresh commands) and every contract regen |
| **Backend tests** | 57 files reference the three status enums | a lifecycle replacement invalidates a large part of the suite; some tests encode behaviour that D-4 … D-42 deliberately removes |
| **Seed builders** | 3 files set statuses directly | **this one gates P0 itself**: the seed is the only source of data in the system (§0), so nothing else can be verified until it runs again |

The seed is the one to notice. It is not "one more caller" — it is the precondition for testing every
other phase, which makes it part of P0 rather than a consequence of it.

---

### P0 — The status enum and the transition graph

The foundation every other phase reads.

- `ObligationStatus` in `app/common/enums.py` beside `ObligationType`, six values (D-39).
- One shared transition graph: forward one stage at a time, events may perform consecutive
  transitions but never skip one (D-35); backward strictly one stage and **always with a reason**
  (D-11, D-33); `submitted` has no outgoing transition; cancel from any unlocked stage (D-23).
- Three enum migrations. VAT is 1:1. Annual merges `not_started`+`collecting_docs` and
  `submitted`+`closed`, and gains an empty stage 2. Advance is a **derivation**, not a mapping —
  `pending` resolves to stage 1 or 2 by whether the turnover is known — and gains stage 4 and
  `canceled`.
- `partial` disappears as a status; a part-paid advance is stage 3 with an outstanding balance (D-7).

**Done when:** one enum, one graph, three domains reading both, and no domain-local status enum left.

### P1 — Delete provable duplication

Unchanged in substance from the pre-re-cut phase of the same name, widened by D-42. Independent of the lifecycle work.

| Item | Change |
|---|---|
| 1.1 | One period-alignment gate. **Both** conditions of `ADVANCE_PAYMENT.INVALID_PERIOD` move to `TAX_CALENDAR` and the code retires (D-3, D-42). The VAT `EXEMPT` check is **not** part of this and stays in VAT — the calendar has no equivalent |
| 1.2 | Delete `bimonthly_advance_payment_period`; rename the survivor `bimonthly_period` |
| 1.3 | Hand-rolled period parsing → `parse_period_year` / `parse_period_month`; the calendar's regex-validating parser becomes the shared implementation |
| 1.4 | One "advances paid for client+year" (the SQL aggregate), one `final_balance` definition |
| 1.5 | `*_resolved_expr()` SQL twins beside each predicate — **and fix the VAT bug**: `_RESOLVED_STATUSES` omits `CANCELED` while `vat_compliance_repository.py:88` excludes it, so a cancelled period reads open on one screen and closed on another |

**Done when:** each rule has one implementation (or one Python + one SQL twin), and `final_balance`
returns the same number from every endpoint that publishes it.

### P2 — Closing and locking

- Closing is a **full lock** in all three: figures, metadata, assignee, notes (D-13). Only VAT
  behaves this way today.
- The closing act records **who** — missing on annual reports and advances (D-13).
- Closing gates generalise into one "what is missing" shape; **assignee required in all three**
  (D-15). Per-domain gates in §4.1.8. Payment in full is not a gate on an advance (D-16).
- "Closed late" recorded at the close; **null**, never `false`, where there is no due date
  (D-20, D-32).
- Annual-report financial lines stop being editable after submission — today the guard checks the
  *client's* status and never the *report's*.

**Done when:** nothing on a closed record can be changed in any of the three, and every closed record
names its author.

### P3 — Amendment and the uniqueness rule

- The compound uniqueness predicate, stated once in §4.1.13: at most one row per client+period that
  is **not deleted, not an amendment, and not cancelled** (D-10, D-22, D-23). Three index changes.
- An explicit "create amendment" action from a closed record — **this does not exist anywhere today**;
  VAT's version is unreachable (§4.1.6).
- An amendment is a full copy of its original, opening at `in_progress` (D-21).
- **A chain is one row everywhere** (D-12). This is the risk item: `sum_net_vat_by_client_record_year`,
  the annual report's VAT import, and the advance turnover lookup would otherwise sum the original
  *and* its amendment. It reaches outside VAT.
- The chain presents the **original's** lateness (D-34).
- An amendment has no due date (D-14) → `due_date_effective` becomes genuinely nullable, every
  overdue query reads NULL as "not late", and `tax_items.py` `_vat_due_date` stops raising on None.

**Done when:** a period can be amended in all three, and no aggregate double-counts a chain.

### P4 — Removal and reconciliation

- Closed obligations become undeletable — today only VAT enforces it; a submitted annual report and a
  paid advance can both be soft-deleted (D-22). This also closes the missing
  `_invalidate_annual_report_tax` on the advance delete path.
- The direct `DELETE` endpoints retire in all three (D-24).
- One reconciliation service, both shapes (§4.2.6): set replacement for VAT and advances, record
  replacement for annual reports. Safety rules from §4.2.4 — past-due rows are debts and are never
  removed, settled rows are reported, removal needs confirmation and is all-or-nothing per year,
  removal is resolved before the creation loop.
- Every "are they all closed?" gate evaluates against the **obligation plan**, not existing rows
  (D-30).
- Removing a worked annual report destroys its lines and the confirmation must say so (D-28).
- Obligation trigger fields become advisor-only (D-36) — `PATCH /clients/{id}` has no role guard at
  all today, and D-24 makes it the only removal path.

**Done when:** an obligation can only disappear as a consequence of correcting a client's
configuration, and only an advisor can cause it.

### P5 — The deadline shape

- Annual reports adopt `due_date_original` / `due_date_effective` / `due_date_override_reason`;
  `filing_deadline` becomes `due_date_effective`. `deadline_type` stays — standard/extended/custom is
  a real annual-only regulatory choice, not a shape difference.
- Drop the legacy `AdvancePayment.due_date` after auditing every consumer, including the several
  `coalesce` expressions in `advance_payment_aggregation_repository.py`.
- One overdue rule over one field name. VAT's work-queue entry moves from `period < now` to
  `due_date_effective` with the shared upcoming window (D-1) — **the one visible product change**:
  VAT periods start appearing before they are late.
- `advance_payment_items` moves from `work_queue/items/billing_items.py` to `tax_items.py`.

**Payoff:** the due-date-override endpoint listed as future work in *both* `docs/domains/vat.md` and
`docs/domains/advance-payments.md` gets built once.

**Done when:** one overdue rule reads one field across all three, and two migrations have landed with
the enum-downgrade convention applied.

### P6 — Domain surgery

The per-domain work that the shared phases do not cover.

**Annual reports** — the signature flow disconnects: `pending_client`, the signature request created
on entering it, the `client_approved_at` gate, and the startup reconciliation (D-5). Readiness drops
4 → 3. `ANNUAL_REPORT_APPROVAL` and the never-referenced `VAT_RETURN_APPROVAL` retire; the
`signature_requests` module stays for engagement agreements, powers of attorney, and custom
documents. `ReportStage` and `POST /{id}/transition` retire (D-40). The forked client-eligibility
check in `annual_report_financial_line_helpers.py` — 403 with `CLIENT.CLOSED`/`CLIENT.FROZEN`, the
codes the shared 409 guard replaced — is removed.

**Advance payments** — turnover gets one source chosen by the client (D-9): the VAT return for VAT
filers, manual entry for those with none, and manual entry is that client's **input event** (D-31).
Retire the refresh commands, `turnover_source`, `turnover_snapshot_at`, `available_turnover`,
`missing_turnover`, and the mismatch filter. `vat_turnover_mismatch_expr` is **narrowed to locked
rows** rather than deleted, becoming the post-lock divergence detector (D-37). A secretary may record
a payment and move stages (D-17). `annual_report_id` is cleared by reconciliation and recorded as a
removal candidate (D-38).

**VAT** — the secretary-can-delete permission is corrected to match D-17.

### P7 — Coupling and arithmetic

- The advance→annual dependency inverts behind an **explicit port** (D-2); no import, at module or
  function level, crosses from `advance_payments` into `annual_reports`.
- VAT auto-populate invalidates persisted `tax_due`/`refund_due` exactly as manual line mutations do.
- `vat_balance` leaves `total_liability` and becomes a separate informational field; the PDF label is
  corrected.

Independent of P3 – P6; may land any time after P2.

### P8 — Generation and rollover

Blocked on **O-1** (rollover policy) and **O-7** (an obligation created for a span the client was not
liable in has no removal route under D-24).

- `annual_obligation_plan(year)` so annual reports answer "is it owed?" the same way.
- One `ObligationGenerationService` covering all three — collapsing `_sync_vat_work_items`,
  `_sync_advance_payments`, and `obligation_orchestrator`, and removing the private
  `_years_to_generate` cross-domain import. This is the same service that reconciles (§4.2.5).
- Office-wide generation covers all three, **reusing** the existing keyset-chunk, per-chunk
  idempotency-key, non-atomic-with-reported-failures design. VAT has no catch-up path of any kind
  today and must have one before real clients are onboarded.
- Implement the rollover policy chosen in O-1.

### P9 — Documentation

Written **from the finished code**, not before it — the order the product owner chose.

- New `docs/domains/tax-lifecycle.md`, `Source of truth: mandatory` (D-41).
- The three domain docs rewritten against the implementation, and their stale claims fixed (§3.4).
- This plan moves to `docs/archive/`.

## 8. Risk summary

Risk is stated **for the pre-production state described in §0**. Every schema and contract row below
gets materially more expensive once real clients exist.

| Phase | Risk now | Risk after launch | Nature |
|---|---|---|---|
| **P0** | medium-low | **very high** | one enum replacing three, across DB, API and frontend. Free of data migration only while §0 holds |
| **P1** | low | low-medium | internal, plus D-42's contract change and both frontend catalogs. Includes one live bug (VAT `CANCELED`) |
| **P2** | low | medium | tightening — three domains gain guards they lack. Nothing loosens |
| **P3** | **high** | high | three index changes, a mechanism that exists nowhere today, and an aggregate correctness risk that reaches outside VAT |
| **P4** | medium | high | retires public endpoints and moves removal onto a reconciliation path; closes a permission leak |
| **P5** | medium-low | **high** | two schema migrations; no backfill or window while §0 holds |
| **P6** | medium | medium | subtractive in the main — a whole signature layer and a whole turnover layer leave |
| **P7** | medium | medium-high | architectural; touches transaction boundaries; no compat surface yet |
| **P8** | medium | medium | consolidates three creation paths; needs O-1 and O-7 |
| **P9** | none | none | documentation |

**P3 is the phase to be careful with.** It is the only one that introduces a mechanism the codebase
has never had, and its failure mode is silent: an aggregate that counts an original and its amendment
produces a wrong number rather than an error — in VAT totals, in the annual report's import, and in
the turnover an advance draws.

## 9. Scope, and what is parked

### 9.1 Scope change — recorded, and resolved

The plan began as a **consolidation** refactor: same behaviour, fewer implementations. The 2026-07-27
specification session turned it into a **lifecycle replacement**, which invalidated the original
phase cut.

**Resolved:** §7 was re-cut on 2026-07-27 against §4.1 – §4.3. Ten phases (P0 – P9) replace the
original seven, and §8's risk table was rewritten with them. The record of the change is kept here
because the two cuts differ in kind, not just in detail — anything written against the old Phase
numbers predates D-4 … D-42.

What grew beyond the original scope, and where it now lives:

| Growth | Phase |
|---|---|
| One status enum replacing three, across DB, API and frontend | P0 |
| Advance payments gain a lock, an amendment mechanism, and two stages | P0, P2, P3 |
| Annual reports lose the signature flow, `ReportStage`, and one readiness gate | P6 |
| A large, mostly subtractive change to the advance-payments turnover layer | P6 |
| Amendment as a real mechanism — it exists nowhere today | P3 |
| Reconciliation as the only removal path | P4 |

### 9.2 Parked — client status

Raised during the same session and deliberately deferred: freezing a client and closing a client are
**byte-identical** today (`client_update_service.py:177-180`), both cancel VAT work items and annual
reports, and both ignore advance payments entirely. The product intent is that freezing should hide a
client's items without destroying them, and only closing should release them. This is a client-domain
change that touches every item type, not only the three tax domains, and is tracked separately.

One finding from it is relevant here regardless: `scope_to_active_clients_stmt` filters **only** on
soft-delete, not on client status, despite its name. Office-wide surfaces do not actually exclude a
frozen client's items — they appear empty only because the items were cancelled.

## 10. Open items not in scope

Recorded so they are not silently absorbed:

- Unverified external tax constants (2026 NI ceiling, 2026 brackets, donation minimum) —
  `docs/domains/annual-reports.md` known issues. Needs the authority's circulars, not a refactor.
- ~~Signature creation running inside the annual-report status-transition transaction.~~
  **Dissolved by D-5**: the annual-report signature flow retires entirely, so the transaction
  boundary it was about ceases to exist. Nothing to fix — the mechanism leaves with P6.
- `AnnualReportDetail.updated_at` nullability — separate known issue.

## 11. What remains — consolidated register

Everything still outstanding, in one place. The detail lives in the sections referenced; this is the
list to read when picking up the work.

### 11.1 Open decisions — 2

| # | Question | Blocks |
|---|---|---|
| **O-1** | **Rollover policy.** Scheduled job, advisor-triggered office-wide generate, or both? Today it is neither reliably: `generate_client_obligations` runs only on client creation and on an obligation-field edit, and there is no scheduler. | P8 |
| **O-7** | **An obligation created for a span the client was not liable in** — a VAT period for 2026-01 on a client registered in 2026-06. The frequency is correct so reconciliation will not touch it, and D-24 removed the direct delete. Either reconciliation also reads a liability start date, or one narrow removal path survives. | P4, P8 |

Nothing else is undecided. O-2 … O-6, O-8 and O-9 closed as D-39 … D-42, D-21, D-23, D-7/D-8.

### 11.2 The frontend — measured, not estimated

No phase in §7 owns this. It is larger than several of them.

> **Partly historical since 2026-07-30.** W2 did this sweep, and the status literals are gone
> from the three feature `contracts.ts` files below — with them the three domain status aliases
> (`VatWorkItemStatus`, `AnnualReportStatus`, `AdvancePaymentStatus`), which every call site now
> replaces with `ObligationStatus` imported bare. Two defects the sweep itself introduced were
> found from the running app rather than from any gate; see the W4 section of the progress doc,
> "an action key is not a status name". The rest of this subsection stands as written.

**Status literals — 14 files hold the old values as strings** (plus `types/generated.ts`, which
regenerates):

| Feature | Files |
|---|---|
| `vatReports` | `constants/vatConstants.ts` · `utils/vatHelpers.ts` · `schemas/workItem.schema.ts` · `api/contracts.ts` · `components/form/VatWorkItemsCreateModal.tsx` · `components/list/VatWorkItemSummaryBar.tsx` |
| `advancedPayments` | `constants.ts` · `api/contracts.ts` · `api/queryKeys.test.ts` |
| `annualReports` | `constants/display.ts` · `api/contracts.ts` · `components/season/seasonProgressConfig.ts` · `utils/panelHelpers.test.ts` |
| **outside the three** | `features/audit/constants.ts` · `features/dashboard/hooks/useSeasonSummary.test.ts` |

The last row is the one to notice: **the status change leaks outside the three features.** Audit and
dashboard both hold the literals, so P0 is not contained by the domains it is about.

**Endpoints that disappear from `endpoints.ts`:**

- `advancedPayments`: `refresh-turnover` (single and bulk) — D-9
- `annualReports`: `amend` — replaced by create-amendment, D-10
- the three `DELETE`s — D-24
- `annual-reports/{id}/transition` — D-40 (not currently wired, so it costs nothing to drop)

**UI that disappears:**

- the turnover refresh control, and the **VAT-mismatch filter with its batch count**
  (`advancedPayments/constants.ts:172`, `messages.ts:327`) — D-9, except the narrowed detector D-37
- the annual-report client-approval surface — D-5
- `partial` as an advance-payment status — D-7

**UI that is new:**

- a create-amendment action, in all three — D-10
- an explicit lock action for advance payments — D-7
- a **reason prompt on every backward transition** — D-33
- a verification stage for advance payments, which has no equivalent today — D-4
- the reconciliation confirmation, including D-28's explicit warning that a worked annual report's
  income and expense lines will be destroyed

**Message catalogs:** 833 lines across the three features (`vatReports` 223, `advancedPayments` 331,
`annualReports` 279), keyed by status and action — the two things this refactor replaces.

### 11.3 Work no phase owns — see §7.3

| Area | Size | Note |
|---|---|---|
| Seed builders | 3 files | **part of P0, not a consequence of it** — the seed is the only source of data (§0), so nothing else can be verified until it runs again |
| Backend tests | 57 files reference the three status enums | some encode behaviour D-4 … D-42 deliberately removes |
| Frontend | §11.2 | unowned |

### 11.4 Parked — separate track

Client status: freezing and closing are byte-identical today, both destroy items, advance payments
are excluded from the cascade entirely, and `scope_to_active_clients_stmt` filters only on
soft-delete despite its name. See §9.2.

### 11.5 Out of scope — 2

Unverified external tax constants (2026 NI ceiling, 2026 brackets, donation minimum) and
`AnnualReportDetail.updated_at` nullability. See §10. A third item — signature creation inside the
status-transition transaction — dissolved with D-5.
