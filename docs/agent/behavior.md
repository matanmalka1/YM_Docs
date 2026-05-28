## Scope
This file owns only:
- Project-wide agent behavior rules.
- How agents communicate, investigate, edit, and report work.

This file must not contain:
- Backend, frontend, API, database, testing, or product/domain rules.

Source of truth: mandatory

# Agent Behavior

- Be direct, critical, and concise.
- Verify existing code, docs, and local patterns before changing structure.
- Do not repeat the user's prompt back to them.
- Do not add greetings, filler, or performative affirmations.
- Make the smallest change that satisfies the task.
- Do not change unrelated files.
- Do not revert user changes unless explicitly requested.
- Before editing, identify which source-of-truth docs apply to the task.
- If a requirement conflicts with project rules, stop and report the conflict before changing affected files.
- If a rule is unclear, flag it instead of guessing.
- If a domain problem is found during generic docs work, list it as an out-of-scope observation only.
- Final responses must state what changed, what was verified, and what remains unresolved.
