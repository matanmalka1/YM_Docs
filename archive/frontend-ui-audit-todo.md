# Frontend UI Audit — TODO

Audit of `frontend/src` (React/TS/Tailwind, RTL Hebrew). 16 code-grounded findings.

## Summary

| # | Severity | Category | Title | Files |
|---|----------|----------|-------|-------|
| 1 | High | State/Table | `onPageSizeChange` dropped — page size unchangeable | PaginatedDataTable.tsx, PaginationCard.tsx, ClientsPage/ChargesPage/UsersPage (+hooks) |
| 2 | High | RTL | `Select` native chevron on wrong side (physical `pr`/`right`) | inputs/Select.tsx |
| 3 | High | A11y/Form | FormField labels a wrapper `<div>`, not the DatePicker button | FormField.tsx, DatePicker.tsx |
| 4 | Medium | UX/Modal | ConfirmDialog autofocuses confirm even for `danger` | ConfirmDialog.tsx |
| 5 | Medium | Design system | Selection uses raw checkboxes; `Checkbox` lacks indeterminate | tableSelection.tsx, Checkbox.tsx |
| 6 | Medium | A11y/Modal | Dialog loses accessible name for non-string `title` | OverlayContainer.tsx, Modal.tsx |
| 7 | Medium | Error handling | Error Alert + "no data" empty state shown together | PaginatedDataTable.tsx |
| 8 | Medium | Data/Form | DatePicker truncates time from datetime values | DatePicker.tsx |
| 9 | Medium | A11y | SelectDropdown highlight invisible to AT (no activedescendant) | SelectDropdown.tsx |
| 10 | Low | Data display | SelectDropdown shows placeholder for unmatched controlled value | SelectDropdown.tsx |
| 11 | Low | RTL | Bulk toolbar Clear positioned with physical `mr-auto` | BulkSelectionToolbar.tsx, GroupedPeriodRow.tsx |
| 13 | Low | A11y/Table | Clickable rows not exposed as interactive | DataTable.tsx |
| 14 | Low | A11y | Alert always `role="alert"` (assertive) for all variants | Alert.tsx |
| ~~15~~ | ~~Low~~ | ~~Design system~~ | ~~Hardcoded Hebrew strings bypass message catalog~~ — DONE | ActiveFilterBadges.tsx, Badge.tsx |
| ~~16~~ | ~~Low~~ | ~~State/Modal~~ | ~~Nested overlays share naive body scroll-lock restore~~ — DONE | useModalDialog.ts |

## TODO

- [x] **#1** Removed the dead `onPageSizeChange`/`pageSizeOptions` props and page-size wiring from `PaginatedDataTable` and Clients/Charges/Users callers.
- [x] **#2** In `Select.tsx` replace `pr-*` → `pe-*` and chevron `right-*` → `end-*`.
- [x] **#3** Reworked `FormField` to pass explicit control props; labels now target the real input/textarea/button controls, and DatePicker/Select trigger buttons receive `id` + `aria-describedby` for errors (`aria-invalid` remains only on controls that support it).
- [x] **#4** Move `autoFocus` to cancel button when `confirmVariant === 'danger'`.
- [x] **#5** Added `indeterminate` prop + ref forwarding to `Checkbox` (forwardRef + useImperativeHandle, sets `el.indeterminate`); replaced both raw `<input type=checkbox>` in `tableSelection.tsx` with `<Checkbox>`.
- [ ] **#6** Use `aria-labelledby` pointing at the title `<h3>` in all OverlayContainer variants.
- [x] **#7** Short-circuit empty/table render when `error && isEmpty` in `PaginatedDataTable`.
- [x] **#8** Decide DatePicker date-only vs datetime contract; preserve time or guard callers.
- [x] **#9** Add listbox/option ids + `aria-controls`/`aria-activedescendant` in `SelectDropdown`. (already present)
- [x] **#10** Distinguish "no value" from "value with no matching option" in `SelectDropdown`. (empty→placeholder, unmatched→raw value; already distinguished)
- [x] **#11** Replace `mr-auto` → `ms-auto` in `BulkSelectionToolbar.tsx` and `GroupedPeriodRow.tsx`.
- [x] **#13** Add interactive `role`/`aria-label` to clickable rows in `DataTable`.
- [x] **#14** Derive `Alert` `role` from `variant` (`status` for non-error/warning).
- [x] **#15** Route hardcoded Hebrew strings through `GLOBAL_UI_MESSAGES`.
- [x] **#16** Extract a ref-counted `useScrollLock`; use in `OverlayContainer` + `ConfirmDialog`.
