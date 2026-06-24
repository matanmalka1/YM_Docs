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
- Use `Card` from `src/components/ui/primitives/Card.tsx` for reusable titled content containers.
  Do not put card-like Tailwind chrome around another component that already renders a `Card`.
- Do not recreate a shared component with local Tailwind markup only to change spacing or color. Add
  a supported variant when the requirement is genuinely reusable.
- When replacing raw markup with a shared primitive, use the primitive's public API first
  (`variant`, `size`, `shape`, `icon`, and similar props). Do not carry over the old raw
  `className` just to preserve pixel-level styling; if the primitive cannot express a recurring need,
  extend the primitive instead of reintroducing local styling.
- A surface that navigates must render a real `Link` (or `ActionSurfaceLink`), not a
  `Button` with `navigate(...)` in `onClick`. Preserve browser link behavior, including opening
  destinations in a new tab and exposing an `href`.
- Keep shared surface variants cohesive: add a variant only after at least two consumers share the
  same structural pattern. Feature-specific animation belongs to the feature, not a generic surface
  variant.
- Use Lucide icons through `lucide-react`. Do not add hand-drawn SVG icons when a matching icon
  exists.
- Use semantic status variants for success, warning, error, and informational states. Do not encode
  product status using arbitrary colors.
- Use `cn()` for conditional classes. Avoid inline styles unless the value is truly dynamic and
  cannot be represented by existing classes or component props.
- The narrow display/layout primitives (`SegmentedControl`, `Chip`/`ChipLabel`, `ActionSurface`,
  `MonoValue`, `DefinitionList`, `Divider`) are scoped deliberately. Convert to one **only** when the
  markup matches its actual shape; do not force-fit. Known boundaries where raw markup is correct:
  - **`MonoValue`** renders a fixed `font-mono text-sm tabular-nums` toned span (5-tone map only). It
    does not fit a `DataTable` column `className:` string (a className slot can't host a component),
    an oversized heading, a cell with child elements, a muted lighter-than-neutral ID, or a span that
    only borrows a tone *color* on otherwise non-mono text (converting would wrongly add monospace).
  - **`DefinitionList`** applies one `valueClassName` to every `<dd>`; per-row value tone goes through
    the `ReactNode` `value` (a pre-styled `<span>`), not a class. It has no label-color hook, so a
    themed label/value block (e.g. warning-colored `<dt>`) stays raw. It uses `border-b` rows, not a
    `divide-y` + muted-row variant — keep a local list when those extra capabilities are needed.
  - **`Divider`** is a standalone sibling rule. A `border-t ... pt-X` that opens a padded
    footer/section is *structural* (the border belongs to that content block) and is not a `Divider`.
  - **`SegmentedControl`** exposes selection via `aria-current`/`aria-pressed`; it does not implement
    the full `role="tab"` roving-tabindex keyboard pattern. A real combobox/listbox or `role="menu"`
    widget with its own keyboard handling is not a SegmentedControl and stays as-is.
  - A primitive is worth *extending* only when ≥2 consumers share the gap; a single lossy caller is
    not justification to widen the primitive — keep that one local and note the divergence.

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
- Select dropdowns must use the shared `Select` component, not a plain `<select>`, when the shared
  component covers the interaction.
- Labeled checkbox controls must use the shared `Checkbox` primitive. Bare native checkboxes are
  allowed only when intentionally unlabeled or hidden as part of another accessible control.
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
- The interactive root element of every shared UI primitive must include `focus-ring`; consumers
  must not need to supply focus styling themselves.
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
