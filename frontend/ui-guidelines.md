## Scope

This file owns only:

- Frontend UI composition, Hebrew/RTL, accessibility, responsive behavior, and interaction rules.
- Rules for using the existing design system and shared UI components.

This file must not contain:

- Product screen specifications, domain behavior, API contracts, or data-fetching architecture.

Source of truth: mandatory

# Frontend UI Guidelines

Read `frontend/DESIGN.md` for the current token catalog and visual reference. This document owns the
implementation rules.

## Design-system usage

- Reuse existing components from `src/components/ui/` before creating a new primitive.
- Reuse `PageHeader`, layout shells, drawers, dialogs, tables, feedback states, and form controls
  consistently across features.
- Do not recreate a shared component with local Tailwind markup only to change spacing or color. Add
  a supported variant when the requirement is genuinely reusable.
- Use Lucide icons through `lucide-react`. Do not add hand-drawn SVG icons when a matching icon
  exists.
- Use semantic status variants for success, warning, error, and informational states. Do not encode
  product status using arbitrary colors.
- Use `cn()` for conditional classes. Avoid inline styles unless the value is truly dynamic and
  cannot be represented by existing classes or component props.

## Hebrew and RTL

- User-facing product copy is Hebrew unless the product term is intentionally English.
- Layouts must work in RTL. Use logical alignment and spacing behavior; do not assume left means
  start or right means end.
- Keep numbers, IDs, email addresses, URLs, dates, and other LTR values readable inside RTL layouts.
  Apply direction locally where needed instead of switching the whole page.
- Icons that communicate direction must match the RTL flow. Icons with fixed semantic meaning, such
  as download, close, edit, or delete, must not be mirrored.
- Truncated Hebrew text must retain an accessible full value through the appropriate label or
  tooltip when the value is operationally important.

## Page states

- Every data-backed page or section must define loading, empty, error, and success states.
- Loading UI should preserve the approximate final layout and avoid major content shifts.
- Empty state copy must distinguish “no records exist” from “no results match the current filters.”
- Error states must include a retry action when retrying can recover the view.
- Do not render `0`, an empty table, or stale content as a substitute for a request that is still
  loading or has failed.

## Forms and actions

- Every input has a visible label or an equivalent accessible name.
- Validation messages appear next to the relevant field. The first invalid field should be reachable
  without hunting through the form.
- Submit buttons show pending state and are disabled against duplicate submission.
- Dirty forms in modals, drawers, or navigable pages must guard against accidental dismissal when
  losing the entered data would be meaningful.
- Destructive actions use the shared confirmation dialog and clearly name the affected record.
- Icon-only actions require an accessible label and a tooltip when the icon is not universally
  obvious.

## Tables, filters, and dense operational UI

- Use the shared table components and column types for operational lists.
- Keep filters, sorting, pagination, and reset behavior consistent with
  `docs/frontend/page-structure.md`.
- Row actions must use the shared row-action pattern. Avoid placing several competing text buttons in
  every row.
- Table loading, no-results, and error states must remain inside the table region so the surrounding
  page does not jump.
- Important values must not rely on color alone. Pair status color with text, icon, or another
  persistent cue.
- Totals displayed above paginated data must come from backend aggregates, not the current page.

## Overlays

- Use a modal for focused creation or confirmation and a drawer for inspecting or editing contextual
  detail without losing the list context.
- Do not stack multiple open overlays unless the workflow has no simpler representation.
- Overlays must close with the shared escape and dismiss behavior, restore focus, and keep keyboard
  focus within the active overlay.
- Overlay content must have a title, a clear primary action, and a clear close/cancel path.

## Accessibility

- Use semantic HTML first. Clickable `div` and `span` elements are not substitutes for buttons and
  links.
- All workflows must be operable by keyboard with a visible focus state.
- Do not disable ESLint accessibility rules to finish a component unless the exception is documented
  and the equivalent accessibility behavior is implemented.
- Form errors, async failures, and success feedback that affect task completion must be available to
  assistive technology.
- Maintain sufficient text and control contrast using the design tokens.

## Responsive behavior

- UI changes must be checked at a desktop width and a narrow mobile width.
- Controls must wrap or collapse intentionally; they must not overlap, clip labels, or force
  horizontal page scrolling.
- Dense tables may use an intentional horizontal scroll region. Keep primary identity and row actions
  discoverable.
- Drawers and modals must fit the viewport and keep their primary actions reachable on small screens.
- Do not hide required functionality on mobile without providing an equivalent path.

## Visual verification

- Browser-check the main success state plus any loading, empty, error, validation, and overlay states
  changed by the task.
- Verify keyboard focus for new controls and confirm that Hebrew text, long values, and narrow widths
  do not overlap or escape their containers.
