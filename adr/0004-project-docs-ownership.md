## Scope
This file owns only:
- The accepted decision about project-wide documentation ownership.

This file must not contain:
- Full documentation structure, workflow rules, or product/domain docs.

Source of truth: historical

# ADR 0004: Project Docs Ownership

Decision: Project-wide documentation must live in root `docs/` or a dedicated docs repo.

Backend and frontend may have thin pointer files only for project-wide agent instructions.

Backend may own backend-specific implementation documentation only.

Frontend may own frontend-specific implementation documentation only.

Backend must not own frontend rules, project-wide agent behavior, or cross-project decision policy.
