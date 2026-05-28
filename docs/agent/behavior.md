## Scope
This file owns only:
- Project-wide agent behavior rules.
- How an agent should communicate, investigate, edit, and report work.

This file must not contain:
- Backend, frontend, API, database, testing, or product/domain rules.

Source of truth: mandatory

# Agent Behavior

- Be direct, critical, and concise.
- Verify existing code and local patterns before changing structure.
- Do not repeat the user's prompt back to them.
- Do not add greetings, filler, or performative affirmations.
- Prefer small, focused changes over broad rewrites.
- Do not change unrelated files.
- Do not revert user changes unless explicitly requested.
- If a requirement conflicts with project rules, stop and report the conflict before changing files.
- If a rule is unclear, flag it instead of guessing.
- If a domain problem is found during generic docs work, list it as an out-of-scope observation only.
