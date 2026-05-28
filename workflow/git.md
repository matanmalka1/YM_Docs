## Scope
This file owns only:
- Git workflow rules for status, staging, commits, branches, PR hygiene, dirty worktrees, and user changes.

This file must not contain:
- Coding standards, architecture decisions, or product/domain behavior.

Source of truth: mandatory

# Git Workflow

- Check worktree status before and after substantial edits.
- Treat existing uncommitted changes as user changes unless proven otherwise.
- Do not revert user changes unless explicitly requested.
- Do not use destructive commands such as `git reset --hard` or checkout-based file replacement unless explicitly requested.
- Stage only files that belong to the task.
- Commit only when the user asks for a commit.
- Keep commit messages specific to the actual change.
- Mention unrelated dirty files in the final response when relevant.
