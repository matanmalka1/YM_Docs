## Scope

This file owns only:

- The remaining (Batch 3) design-system primitive-API follow-up items from the 2026-06-28 audit.
- Locked user decisions to carry into execution.

This file must not contain:

- Completed work (lives in code + `docs/archive/design-system-audit-plan.md`).
- Architecture/UI rules (those belong in `docs/frontend/ui-guidelines.md` once a change lands).

Source of truth: tracking only — not source of truth for current behavior.

# Design-System Follow-up — Batch 3 (remaining)

Backlog only. Batch 1 (safe) and Batch 2 (additive primitive props) from the original audit are **done**
in code; the full historical catalog is archived at `docs/archive/design-system-audit-plan.md`. These
remaining items are API-surface / taste changes — open as deliberate work, not a mechanical sweep.

Risk legend: 🟡 additive (no caller breaks) · 🔴 decision/visual/taste.

## Items

### F10 — `Button` size scale too narrow; no icon-only mode · 🔴

- Sizes `sm|md` both `text-sm`; callers fake small with `className="text-xs"`. No square/icon-only mode.
- **Fix:** add `xs` size (`text-xs`, tighter padding); optionally `iconOnly?: boolean` (square padding,
  requires `aria-label`).
- **text-xs faker call sites → `size="xs"`:** `features/search/components/SearchFiltersBar.tsx:109`,
  `features/annualReports/components/financials/FinancialLineFormParts.tsx:20`,
  `features/annualReports/components/panel/ReportAlertBanners.tsx:47`.
- **icon-only (broad; optional):** `features/binders/components/sections/BinderDocumentsSection.tsx:73`
  and most RowActions icon buttons.

### F11 — StatsCard variant → semantic tones · 🔴 **FULL RENAME (locked)**

- **DECISION (locked):** change `StatVariant` from color-names to `info|positive|negative|warning|purple|neutral`
  (map blue→info, green→positive, red→negative, orange→warning; keep `purple` = sanctioned violet accent,
  keep `neutral`).
- Then migrate **all ~16** stat-section helpers' `variant: '<color>' as const` literals to the tone names.
- **Grep entry point:** `grep -rn "variant: '" features/**/*StatsSection*` + every `<StatsCard>` caller
  (clients/charges/workQueue/annualReports/taxDashboard/binders/advancedPayments/reports …).
- Update memory `reference_design_system_source_of_truth` if the naming convention is documented.

### F13 — `StatusBadge` drops Badge's `dot`/`ring`/`onClick` · 🟡

- **Fix:** forward `dot?`/`ring?`/`onClick?` (or accept `badgeProps`). Additive — no breaks (~13 callers
  across clients/tasks/charges/vatReports/binders/signatureRequests/advancedPayments).

### F14 — `Tooltip.text` / `Alert.message` are `string`-only · 🟡

- **Fix:** widen `Tooltip.text` (`Tooltip.tsx:16`) and `Alert.message` (`Alert.tsx:5`) to `React.ReactNode`;
  add `Tooltip` `placement?`. Decouple `Alert.onRetry` styling from the negative/red palette
  (`Alert.tsx:83-91`) so a warning-alert retry isn't red.
- **Alert `onRetry` callers (5):** `tasks/.../ClientTasksTab.tsx`, `tasks/.../TasksListPanel.tsx`,
  `tasks/pages/TasksPage.tsx`, `authorityContacts/.../AuthorityContactsCard.tsx`,
  `authorityContacts/.../AuthorityContactsListCard.tsx`. Additive — no breaks.

### F15 — `DatePicker`: no `minDate`; `compact` boolean vs `size` convention · 🟡/🔴

- **Fix:** add `minDate?: Date` (mirror `maxDate`). Optionally rename `compact` → `size?: 'sm'|'md'` for
  cross-primitive consistency.
- **`compact` caller (only):** `features/users/components/detail/AuditLogsDrawer.tsx:76,77`. `minDate` additive.

### F16 — `DrawerField` duplicates `DefinitionList` (stacked) · 🔴 **undecided — confirm appetite first**

- `DrawerPrimitives.tsx:8-13` ≈ `DefinitionList.tsx:47-54` — two APIs for one layout.
- **Fix:** consolidate `DrawerField` onto `DefinitionList layout="stacked"` (or make DrawerField a thin
  re-export). Migrates **7 callers**: `invoices/.../ChargeInvoiceSection.tsx`, `charges/.../ChargeDetailDrawer.tsx`,
  `binders/.../BinderDetailsPanel.tsx`, `signatureRequests/.../SignatureRequestAuditDrawer.tsx`,
  `advancedPayments/.../AdvancePaymentDrawer.tsx`, `advancedPayments/.../AdvancePaymentReadonlySections.tsx`,
  `notifications/.../NotificationDetailDrawer.tsx`.

### F17 — `TimelineEntry` dot color hardcoded (no `tone`) · 🟡

- **Fix:** add `tone?: 'default'|'success'|'warning'|'error'` on `TimelineEntry` (`Timeline.tsx:32`).
- **Callers:** `features/timeline/components/*`, `binders/.../BinderAuditSection.tsx`,
  `binders/.../BinderIntakesSection.tsx`, `users/.../AuditLogsDrawer.tsx`,
  `signatureRequests/.../SignatureRequestAuditDrawer.tsx`,
  `annualReports/.../{FilingTimelineTab,TimelineEvent,StatusAuditTimeline,UpcomingDeadlinesList}.tsx`. Additive.

### F19 — Minor API smells · 🔴 accept / low-value

- `Badge.removable` ignores `variant` (always primary palette) — no removable error/warning badge
  (`Badge.tsx:86-108`).
- `Badge.dot` takes a raw class string (leaky) (`Badge.tsx:13`).
- `MonoValue` uses `tone` (naming drift vs `variant`) and bakes day-thresholds 60/90 into a generic
  primitive (`MonoValue.tsx:22-27`) — consider a feature wrapper / `thresholds` prop. Callers:
  `vatReports/.../VatPeriodCard.tsx`, `binders/.../BinderDetailsPanel.tsx`, `binders/.../BindersColumns.tsx`,
  `annualReports/.../AnnualReportVatAutoPopulateResultPanel.tsx`.
- `GroupSection.collapsible` uncontrolled-only (1 caller:
  `annualReports/.../AnnualReportOverviewSection.tsx`).
- Cross-cutting size-scale divergence (`xs|sm|md` vs `sm|md` vs `sm|md|lg`): a shared `Size` type would
  surface gaps but is a broad refactor — accept/track, do not force now.

## Verification (per change)

- `npm run typecheck`, `npm run lint`, `npm run arch:check`, `npm run test`.
- Hebrew RTL visual diff on touched surfaces; `npm run build` to confirm new utility classes emit.
- Re-sync the static `design-system/` demo for any new visual token/size/tone class.
- When a change lands, document the convention in `docs/frontend/ui-guidelines.md` and drop the item here.
