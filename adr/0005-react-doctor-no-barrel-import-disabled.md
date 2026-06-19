## Scope
This file owns only:
- The accepted decision to disable react-doctor's `no-barrel-import` rule for this frontend.

This file must not contain:
- General react-doctor configuration, other rule decisions, or build tooling rules.

Source of truth: historical

# ADR 0005: Disable react-doctor `no-barrel-import`

Decision: Disable `react-doctor/no-barrel-import` in `frontend/doctor.config.json`.

Context:

- `no-barrel-import` flags importing from any local `index.ts` that only re-exports, telling each
  caller to deep-import the leaf module. It fired on 128 frontend files (mostly intra-feature
  `../api` → `../api/queryKeys`).
- This directly contradicts the documented architecture (`docs/frontend/architecture.md`,
  Dependency boundaries): cross-feature imports **must** go through the owning feature's public
  `index.ts`, and feature sub-barrels like `../api` are explicitly sanctioned. `arch-check --strict`
  enforces this and would report `BARREL_BYPASS` for the cross-feature rewrites the rule wants.
- The stack is Vite. Production builds tree-shake barrel re-exports, so the rule's stated harm
  ("ships extra code, slows page load") does not apply to the production bundle. The only real cost
  is a marginally slower dev-server cold import, which does not justify rewriting ~126 files.

Consequences:

- The rule stays off; barrels remain the intended module boundary.
- If barrel cold-start ever becomes a measured dev-server problem, address it with Vite
  config/warmup, not by deep-importing across the codebase.
- A one-off codemod was attempted and reverted; no import rewrites were kept.
