## Scope
This file owns only:
- Documentation ownership rules.
- How agents decide which file is authoritative for a topic.

This file must not contain:
- The actual backend, frontend, API, database, workflow, or product/domain rules.

Source of truth: mandatory

# Source Of Truth

- Project-wide rules live in root `docs/` or a dedicated docs repo.
- Backend docs may own backend-specific implementation details only.
- Frontend docs may own frontend-specific implementation details only.
- Backend must not own frontend rules, project-wide agent behavior, or cross-project decision policy.
- Root/backend/frontend `AGENTS.md` and `CLAUDE.md` files are thin pointers only.
- Product/domain behavior belongs in product/domain docs, not architecture docs.
- Temporary TODOs must not be mixed into permanent architecture rules.
- ADRs own accepted historical decisions and their rationale.
- When in doubt, prefer code and tests, then update docs to match the verified behavior.
- If docs and code disagree, verify the code, report the drift, and update the relevant source-of-truth doc as part of the change when in scope.
