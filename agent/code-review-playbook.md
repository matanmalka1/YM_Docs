## Scope
This file owns only:
- The review-for-correctness checklist agents apply when reviewing a code or docs change before approval.

This file must not contain:
- Completion/verification steps (those live in `docs/workflow/verification.md`).
- Full security or backend-architecture rules (link to the owners instead).

Source of truth: mandatory

# Code Review Playbook

Apply this when **reviewing a code or docs change before approving it**. It catches correctness/logic regressions that tests and completion checks miss.

This is not the completion checklist: build/typecheck/test/OpenAPI/migration/docs-updated gating stays in `docs/workflow/verification.md`. Use that to confirm a change is *finished*; use this to confirm a change is *correct*.

## 1. Recurring-findings checklist

Actively check each pattern before concluding "no issues found". For each finding, record what is wrong, the `path:line`, the rule/invariant it violates, and a suggested fix.

| # | Pattern | What to check | Owner rule |
|---|---------|---------------|------------|
| 1 | IDOR / missing ownership re-check | Compare each `update_*` / `read_*` / `delete_*` path against its `create_*` counterpart. If create validates that a child/related id (line, invoice, activity, document) belongs to the parent/owner, the update/read/delete path MUST re-check the same. Mutating or reading by child `id` after only checking the parent exists is a finding. | `docs/architecture/security.md` |
| 2 | Documented-but-unenforced invariant | For each domain invariant (e.g. "no transition to X without field Y"), confirm the service actually enforces it. Documented but not enforced in code = finding. | `docs/backend/architecture.md` (service layer) |
| 3 | Computed field reading a stale/legacy source | When a field has both a legacy column and a newer source-of-truth column, confirm computed/derived values read the new one. | owning domain doc |
| 4 | Error code off `DOMAIN.REASON` format | Grep raised codes; flag any not matching `DOMAIN.REASON` and any unregistered namespace. | `docs/backend/error-codes.md`, `docs/architecture/api-contracts.md` |
| 5 | Broken/stale imports & dead references | Imports of symbols not defined at the target; module/doc references to files or paths that no longer exist (including stale doc links after relocations). | — |

## 2. Layering & API contract checks

| Check | Expectation | Owner rule |
|-------|-------------|------------|
| Router responsibility | Routers parse the request, apply dependencies, call one service, map the response. No business branching, orchestration, or persistence. | `docs/backend/architecture.md` |
| Service responsibility | Services own business logic, orchestration, derived state, fine-grained authorization, and invariants; raise `AppError` subclasses, not `HTTPException`. | `docs/backend/architecture.md` |
| Repository responsibility | Repositories own query/persistence only — no business or authorization decisions, no cross-domain orchestration, no raw SQL, no Pydantic construction. | `docs/backend/architecture.md` |
| API contract changes | URL/list/filter/sort/error-envelope/DTO changes match the binding contract. | `docs/architecture/api-contracts.md` |
| PATCH / update semantics | `*UpdateRequest` follows empty-PATCH 422, unknown-field rejection, and explicit-null rules. | `docs/architecture/update-request-conventions.md` |

## 3. What this playbook is not

- Not a testing checklist — see `docs/backend/testing.md` / `docs/frontend/testing.md`.
- Not a release/completion checklist — see `docs/workflow/verification.md`.
- Not a replacement for the security, backend-architecture, or domain docs — it links to them; those remain the source of truth.
