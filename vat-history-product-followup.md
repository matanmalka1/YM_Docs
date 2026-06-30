## Scope

This file owns only:

- A product follow-up item for reviewing the VAT work-item History tab after the audit refactor.

This file must not contain:

- Canonical VAT/audit behavior rules.
- Architecture rules or endpoint contracts.
- Implementation work already accepted as current behavior.

Source of truth: tracking only — not source of truth for current behavior.

# VAT History — Product Follow-up

Backlog only. Captures a product/UX check created during the audit refactor Phase 3 review.

## Context

Phase 3 moves VAT audit from the legacy `VatAuditLog` surface to generic entity audit trails.

The intended entity split is:

- `vat_work_item` trail: work-item lifecycle and work-item-level changes.
- `vat_invoice` trail: invoice add/update/delete and invoice-level amount changes.

That means invoice events now live on the `vat_invoice` audit trail. They no longer appear in the
VAT work-item History tab if that tab reads only:

```text
/audit/vat_work_item/{work_item_id}
```

This is the intended audit-model split: each audit row belongs to the entity that actually changed.

## Product question

Should the VAT work-item History tab remain a strict work-item trail, or should the product expose an
aggregated VAT-period history view that combines:

- `/audit/vat_work_item/{work_item_id}`
- `/audit/vat_invoice/{invoice_id}` for all invoices under that work item

## Current expected behavior

If the History tab remains bound only to `vat_work_item`, it should show work-item events such as:

- `vat_work_item.created`
- `vat_work_item.status_changed`
- `vat_work_item.filed`
- `vat_work_item.amount_overridden`
- `vat_work_item.deleted`

It should not show invoice events such as:

- `vat_invoice.created`
- `vat_invoice.updated`
- `vat_invoice.amount_changed`
- `vat_invoice.deleted`

Those events are available from the relevant `vat_invoice` trail.

## Review criteria

During full product review, check:

1. Whether users expect the VAT work-item History tab to answer “what happened to this VAT period?”
   or only “what happened to this work-item record?”
2. Whether invoice-level events disappearing from that tab creates a practical traceability gap.
3. Whether invoice rows/details should expose their own history affordance.
4. Whether an aggregated VAT-period history endpoint/view is needed, instead of overloading the
   generic entity-audit endpoint.
5. If aggregation is needed, define it explicitly as a product timeline/history view, not by weakening
   the entity-audit ownership model.

## Candidate outcomes

### Option A — Keep strict entity history

The work-item History tab stays scoped to `vat_work_item`. Invoice history is available only from
invoice-specific UI.

This keeps the audit model simple and exact.

### Option B — Add invoice history affordance

Keep the work-item History tab strict, but add a per-invoice history drawer/action that reads:

```text
/audit/vat_invoice/{invoice_id}
```

This preserves entity audit while making invoice changes discoverable.

### Option C — Add aggregated VAT-period history

Add a new product-level history view that combines work-item and child-invoice audit rows for a VAT
period.

This should be a deliberate aggregation layer. It must not reintroduce the deleted legacy
`/vat/work-items/{id}/audit` route as a compatibility wrapper.

## Recommendation for review

Start by testing whether Option A creates confusion in real VAT review flows. If users need one place
to answer “what changed in this VAT period?”, prefer Option C as a new explicit product view. If the
need is mostly invoice-level traceability, prefer Option B.
