## Scope
This file owns only:
- Git workflow rules for status, staging, commits, branches, PR hygiene, dirty worktrees, and user changes.

This file must not contain:
- Coding standards, architecture decisions, or product/domain behavior.

Source of truth: mandatory

# Git Workflow

- The project is split into three git repositories: `backend/`, `docs/`, and `frontend/`.
- The project root is not the git repository for code changes; run git commands from the relevant
  repo directory or use `git -C backend`, `git -C docs`, or `git -C frontend`.
- Check worktree status before staging and before finishing.
- Treat existing uncommitted changes as user changes unless proven otherwise.
- Do not revert user changes unless explicitly requested.
- Do not use destructive commands such as `git reset --hard` or checkout-based file replacement unless explicitly requested.
- Stage only files that belong to the task.
- Commit only when the user asks for a commit.
- Keep commit messages specific to the actual change.
- Do not mix unrelated concerns in the same commit.
- Mention unrelated dirty files in the final response when relevant.
