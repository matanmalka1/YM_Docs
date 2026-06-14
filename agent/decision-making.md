## Scope
This file owns only:
- Cross-project technical decision rules.
- Defaults for choosing between competing implementation approaches.

This file must not contain:
- Product/domain behavior, module-specific workflows, or temporary implementation plans.

Source of truth: mandatory

# Decision-Making

- Choose the simplest maintainable design that fits the existing architecture.
- Use existing local patterns unless they conflict with mandatory docs.
- Add an abstraction only when it removes real duplication or clarifies ownership.
- Do not preserve legacy behavior unless explicitly requested.
- Do not add aliases, wrappers, compatibility shims, or alternate names to avoid updating callers.
- Do not create hidden fallback behavior.
- Remove duplicated logic instead of adding another copy.
- Before adding literals or new constants, search for existing project-wide and feature-owned constants and reuse one only when its semantic meaning matches.
- Apply a shared constant consistently across request parameters, comparisons, calculations, and user-facing text that represent the same value.
- Make configuration fields required when every valid entry needs them; do not use optional fields when omission would silently hide or disable behavior.
- Before renaming URL parameters, query inputs, configuration fields, or other shared contracts, trace and update every producer and consumer; do not assume a new name is compatible.
- Say when existing structure is wrong and explain the operational impact.
- Local mandatory architecture rules override generic best practices.
- Treat the system as in development unless the user explicitly says production users or data are involved.
