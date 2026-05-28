## Scope
This file owns only:
- The accepted decision about hidden fallback behavior.

This file must not contain:
- Product-specific fallback decisions or temporary rollout plans.

Source of truth: historical

# ADR 0003: No Hidden Fallbacks

Decision: Do not add hidden fallback behavior.

Consequences:

- Missing data, unsupported states, and contract mismatches should fail clearly.
- Agents must not hide errors by silently trying unrelated sources.
- If fallback behavior is required, it must be explicit, documented, and tested.
