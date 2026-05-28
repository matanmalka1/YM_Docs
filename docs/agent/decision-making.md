## Scope
This file owns only:
- Cross-project technical decision rules.
- Defaults for choosing between competing implementation approaches.

This file must not contain:
- Product/domain behavior, module-specific workflows, or temporary implementation plans.

Source of truth: mandatory

# Decision-Making

- Prefer simple, maintainable architecture.
- Prefer the current codebase's established patterns over new abstractions.
- Add abstractions only when they remove real duplication or clarify ownership.
- Do not preserve legacy behavior unless explicitly requested.
- Do not add aliases, wrappers, compatibility shims, or alternate names to avoid updating callers.
- Do not create hidden fallback behavior.
- Avoid duplicated logic across backend, frontend, and docs.
- Be willing to say when existing structure is wrong.
- Modern backend/frontend practices are preferred, but local architecture rules win.
- This project is in development unless explicitly stated otherwise; do not assume production users or production data.
