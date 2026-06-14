## Scope
This file owns only:
- How frontend tests are written and run.

This file must not contain:
- Broad completion criteria, manual QA checklist, permanent architecture rules, or domain acceptance criteria.

Source of truth: mandatory

# Frontend Testing

- Choose test scope according to behavioral risk and the surface changed.
- Batch routine verification at a stable checkpoint after a coherent set of edits. During iteration,
  use a focused test when it provides useful feedback for the next decision.
- Run the full suite when the change is broad, shared, risky, or explicitly requested.
- Do not claim tests passed unless they were run in the current task.
- See `docs/frontend/commands.md` for the canonical frontend test commands.
- Frontend tests use Vitest and live beside the code as `*.test.ts`, `*.test.tsx`, `*.spec.ts`, or
  `*.spec.tsx`.
- Add or update focused tests when changing pure business/display helpers, query-param serialization,
  schemas, endpoint construction, cache-update logic, or a bug with a stable regression case.
- Test behavior and public outputs. Do not assert component implementation details, internal hook
  state, or Tailwind class strings unless the class is the behavior under test.
- API tests must verify request method, endpoint, query params, payload, and response mapping where
  those contracts changed.
- URL-state tests must cover parsing, omission of defaults, invalid values, reset behavior, and
  preservation of unrelated search params where applicable.
- Form/schema tests must cover valid values, required fields, enum rejection, nullability, and any
  UI-to-API normalization.
- Mutation tests should verify the narrow cache update or invalidation behavior when that logic is
  non-trivial.
- A UI-only visual change does not require a synthetic unit test when browser verification proves the
  change more directly. Report the browser states and widths checked.
- Prefer the smallest relevant Vitest scope for focused behavior. Use `npm run test` at the completion
  checkpoint when shared helpers, shared UI, routing, auth, or cross-feature behavior changed.
- If frontend test coverage or required browser tooling is missing for an area, report that honestly
  instead of inventing test results.

## Browser tooling in IDE sessions

- When Codex runs from an IDE session, the in-app Browser may be listed as available but still have no
  active browser instance. If connecting reports that the in-app Browser is unavailable, report that
  limitation and do not claim browser QA was completed.
- Use the Chrome plugin from IDE sessions only after verifying that its browser connection works. If
  Chrome is installed and the Codex Chrome Extension is enabled but the plugin still cannot connect,
  report the Chrome connection failure instead of treating Chrome as the verified browser path.
