## Scope
This file owns only:
- Archived historical content from the original work-queue backend domain document (`backend/docs/backend/domains/work-queue.md`).

This file must not contain:
- Current implemented behavior (see canonical doc).
- New product requirements.

Source of truth: historical

# Work Queue Domain (Archived)

> Original backend-domain document. Not the current canonical contract.
> Canonical domain doc: `docs/domains/work-queue.md`

**What was dropped and why:**
- Inline pseudo-code and repeated restatements of source builders were compressed because the canonical doc now cites the real implementation paths and line ranges.
- Frontend-oriented wording such as "Used by the frontend for stable React keys" was reduced to backend-contract language.
- Generic performance commentary was kept only where it still describes current in-memory behavior; it no longer acts as design guidance.

**What was preserved (with rationale):**
- The queue is a computed, non-persisted view. This is still true and central to the domain.
- `task` remains the only manual source type; all other source types are derived from other domains.
- Open linked tasks still merge into system rows, while other tasks remain standalone.
- Summary is still computed on the full filtered set before pagination.
- The old scoping notes still matter historically because they explain why some source types are client-level and why binders are only exposed on the global queue.

---

## Historical summary

The original document described the work queue as a unified in-memory list of office work items built from:

- `vat_work_item`
- `annual_report`
- `advance_payment`
- `charge`
- `binder`
- `task`

It documented the single public entry point as `GET /api/v1/work-queue`, the build flow of:

1. building system rows from source domains,
2. attaching source actions,
3. loading tasks,
4. merging open linked tasks into source rows,
5. filtering,
6. sorting by urgency and due date.

It also recorded the historical scoping rule that:

- `client_record_id` narrows all source types to one client,
- `business_id` narrows only charge items,
- binder rows appear only when the queue is fully unscoped.

The warning categories preserved from that document were:

- `source_missing`
- `source_final`
- `source_unknown`
- `multiple_linked_tasks`

The legacy doc also explicitly stated that there is no `work_queue` table and that queue item ids are synthetic strings such as `vat_work_item:42` and `task:7`.
