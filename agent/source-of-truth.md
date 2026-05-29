## Scope
This file owns only:
- Documentation ownership rules.
- How agents decide which file is authoritative for a topic.

This file must not contain:
- The actual backend, frontend, API, database, workflow, or product/domain rules.

Source of truth: mandatory

# Source Of Truth

- Project-wide rules must live in root `docs/` or a dedicated docs repo.
- Backend docs may own backend-specific implementation details only.
- Frontend docs may own frontend-specific implementation details only.
- Backend must not own frontend rules, project-wide agent behavior, or cross-project decision policy.
- Root/backend/frontend `AGENTS.md` and `CLAUDE.md` files must be thin pointers only.
- Product/domain behavior must live in product/domain docs, not architecture docs.
- Temporary task lists must not be mixed into permanent architecture rules.
- ADRs own accepted decisions and rationale.
- When docs and code disagree, verify code and tests first, then report the drift.
- Update the relevant source-of-truth doc in the same task when the requested change intentionally changes a rule.
- Do not duplicate a rule in multiple files; replace duplicates with a pointer to the owning file.
