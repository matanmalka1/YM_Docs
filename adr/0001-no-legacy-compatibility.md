## Scope
This file owns only:
- The accepted decision about legacy compatibility.

This file must not contain:
- General coding standards, module-specific migration plans, or product/domain behavior.

Source of truth: historical

# ADR 0001: No Legacy Compatibility By Default

Decision: Do not preserve legacy compatibility unless explicitly requested.

Consequences:

- Remove old callers instead of adding aliases.
- Do not add compatibility wrappers to avoid updating code.
- Do not keep hidden fallback behavior for old contracts.
- If compatibility is requested, document the reason, scope, and removal plan.
