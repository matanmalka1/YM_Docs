## Scope

This file owns only:

- The reusable workflow + prompt template for authoring one canonical domain doc under `docs/domains/`.
- Parallel-safety rules so two agents (Codex, Claude Code) can each take a different domain without conflicts.

This file must not contain:

- The domain docs themselves.
- Architecture/API rules (those live in `docs/architecture/*`).

Source of truth: mandatory

# Domain Docs Authoring

Goal: produce one canonical, code-accurate doc per backend domain at `docs/domains/<DOMAIN>.md`, cleaning legacy/stale/incorrect material from the old `backend/docs` files into an archive.

Two agents run this in parallel, each on a **different** domain. The template below is parameterized; fill the variables, then follow the procedure exactly.

## Variables

- `{DOMAIN}` — kebab-case doc name, e.g. `advance-payments`, `vat-reports`, `tax-calendar`.
- `{MODULE}` — backend Python module dir under `backend/app/`, e.g. `advance_payments`, `vat_reports`, `tax_calendar`. Derive it by listing `backend/app/*/api`; if unsure, confirm before writing.
- `{LEGACY_PATHS}` — the existing `backend/docs/**` files for this domain (may be zero). Find with a grep for the domain name/module.

## Parallel-safety rules (read first)

1. **Claim one domain.** Before starting, state which `{DOMAIN}` you are taking. Do not touch any other domain's files.
2. **Touch only your own files:**
   - `docs/domains/{DOMAIN}.md` (create)
   - `docs/archive/{DOMAIN}-legacy.md` (create, only if there is legacy material worth keeping)
   - Your `{LEGACY_PATHS}` under `backend/docs` (replace body with a pointer — see Archiving)
3. **Do NOT edit shared/index files.** Specifically do not edit:
   - `docs/domains/README.md`
   - `docs/project/documentation-map.md`
   - `docs/archive/README.md`
   - any other domain's doc

   Registration into the index/map is batched separately by a human after parallel runs finish, to avoid merge conflicts. Instead, end your run with a **Registration block** (see Output) listing the one-line index/map entries you would have added.

4. **Docs-only.** Never change code, endpoints, schemas, models, or migrations. If code looks wrong, report it — do not "fix" the doc to hide it.
5. **No deletes.** Never delete a file. Legacy files are emptied to a pointer; their content moves to the archive copy.

## Authoring procedure

1. **Gather authoritative facts from code** (`backend/app/{MODULE}`):
   - Endpoints: routers under `{MODULE}/api/` (include exact method + path + router `prefix`). Cross-check against `backend/openapi.json` as the primary contract.
   - Model + fields: `{MODULE}/models/`. List real columns, nullability, and FKs.
   - Enums/statuses: `{MODULE}/models/*enum*` or model files.
   - Domain rules/invariants: `{MODULE}/services/`.
   - Error codes: grep `"{DOMAIN_UPPER}\\."` in `{MODULE}`; reference `docs/backend/error-codes.md`.
2. **Preserve domain decisions (hybrid).** From `{LEGACY_PATHS}` and `backend/docs/domain_decisions_v3.md`, keep decisions/rationale that are still true. Drop anything contradicted by code. Mark anything not yet implemented as `Future / planned`.
3. **Verify every claim.** For each non-obvious fact, confirm against a `path:line`. Endpoints must exist in `backend/openapi.json`. Do not invent behavior. Before writing, skim existing domain docs for recurring Known issues patterns and check whether your domain has the same (see Known issues patterns below).
4. **Write** `docs/domains/{DOMAIN}.md` using the skeleton below.
5. **Archive legacy.** See Archiving.

## Conflict / precedence

Follow `docs/project/documentation-map.md`: ADR > architecture > workflow > project > existing code. The new domain doc is a **project/domain doc** — it must not restate or override architecture rules; link to them instead (e.g. API envelope → `docs/architecture/api-contracts.md`).

## Archiving legacy

For each file in `{LEGACY_PATHS}` that has historically useful material:

1. Copy the still-useful historical content to `docs/archive/{DOMAIN}-legacy.md` (add the standard Scope header, `Source of truth: historical`).
2. Replace the body of the original `backend/docs` file with a short pointer:
   > Historical. Canonical domain doc: `docs/domains/{DOMAIN}.md`. Archived original: `docs/archive/{DOMAIN}-legacy.md`.

If a legacy file has nothing worth keeping, just replace its body with the pointer (no archive copy).

## Link convention

All cross-doc links use a **single** `docs/` prefix (relative to the docs repo root), e.g. `docs/backend/error-codes.md`, `docs/archive/{DOMAIN}-legacy.md`. Never write `docs/...` — that is the monorepo-root vantage and breaks the project convention used by `documentation-map.md` and every existing doc. The only files that legitimately use `docs/...` are the pointer files outside the docs repo (`backend/docs/**`, root/backend `CLAUDE.md`).

## Canonical doc skeleton

Use exactly these sections (omit a section only if truly N/A, and say why):

```markdown
## Scope

This file owns only:

- Canonical current-state documentation for the {DOMAIN} domain.

This file must not contain:

- Architecture/API rules (link to docs/architecture/\*).
- Other domains' behavior.

Source of truth: mandatory

# {Domain Title}

One-paragraph summary: what this domain represents and its core responsibility.
Last verified against code + backend/openapi.json: {YYYY-MM-DD}.

## Endpoints

| Method | Path        | Purpose |
| ------ | ----------- | ------- |
| ...    | /api/v1/... | ...     |

(All paths must exist in backend/openapi.json.)

## Model & fields

Key model(s) and their real columns. Note nullability and FKs. Cite `backend/app/{MODULE}/models/...:line`.

## Enums / statuses

The real enum values from code (exact spelling). Cite the source file.

## Domain rules & invariants

Bullet the rules actually enforced in services. Cite `services/...`. Mark anything aspirational as `Future / planned`.

## Error codes

The `DOMAIN.REASON` codes this domain raises. Registry: `docs/backend/error-codes.md`.

## Known issues

Current code discrepancies found during authoring (bugs to fix in code, not intended behavior). Each item: what is wrong, `path:line`, the rule/invariant it violates, and a suggested fix. Omit the section if none found. Docs-only — never fix the code here; report it.

Actively check for these recurring patterns before concluding "none found":

1. **IDOR / missing ownership re-check on update.** Compare each `update_*`/`delete_*` service path against its `create_*` counterpart. If create validates that a child/related id (line, invoice, activity, document) belongs to the parent/owner, the update/delete path MUST do the same. A path that mutates by child `id` after only checking the parent exists is a finding. (Seen in annual-reports F-001, vat-reports F-008.)
2. **Enforced invariants that the service skips.** For each invariant in the domain decisions (e.g. "no transition to X without field Y"), grep the service to confirm it is actually enforced. Documented-but-unenforced = finding. (Seen in vat-reports F-007 assigned_to.)
3. **Computed/derived fields using a legacy/stale source.** When a field has both a legacy column and a newer source-of-truth column, confirm computed values read the new one. (Seen in advance-payments F-005 due_date vs due_date_effective.)
4. **Error codes off the `DOMAIN.REASON` format.** Grep raised codes; flag any that are not `DOMAIN.REASON`. (Seen in vat-reports F-009.)
5. **Broken/stale imports & dead references.** Imports of symbols not defined at the target, and module/README references to files that no longer exist. (Seen in businesses F-006.)

Record each finding in this section and report it in the task summary.

## Decisions (preserved)

Still-true domain decisions carried over from legacy specs, with rationale. Drop anything code contradicts.

## Future / planned

Explicitly not-yet-implemented behavior. Never describe as current.

## Historical notes

Pointer to `docs/archive/{DOMAIN}-legacy.md` if archived.
```

## Required output / report

End every run with:

1. **Files changed** — list.
2. **Verification** — endpoints confirmed against `openapi.json`; key facts with `path:line`.
3. **Stale/legacy removed** — what was dropped and why.
4. **Future / planned** — what was marked aspirational.
5. **Could not verify** — any unverifiable claims.
6. **Registration block** — the exact one-line entries to add later to `docs/domains/README.md` and `docs/project/documentation-map.md` (do not add them yourself).
