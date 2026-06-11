# LIST DTO payload reduction

Task 34 — split fat double-duty DTOs into thin list DTOs (LIST) and the
existing full DTOs (DETAIL).

## Method

The four affected LIST endpoints already returned enriched objects, so a clean
"before" cannot be captured against the live endpoint after the refactor
without reverting. Instead we measured **the response models directly** over an
identical, fully-populated record:

- Build one fully-populated instance of each fat `…Response` DTO ("before").
- Build the thin `…ListItem` from the same instance via `model_validate`
  ("after").
- Serialize both with Pydantic `model_dump_json()` (the same serializer FastAPI
  uses) and compare UTF-8 byte length.
- Scale to a `page_size=50` page (`bytes/row × 50`) for the per-page figure.
  Wrapper/envelope bytes (`total`, `page`, `page_size`, `stats`) are constant
  and excluded, so the table is a like-for-like comparison of the row payload.

This avoids fabricated numbers: every byte count is the real serializer output
for the same data, fat vs thin.

Reproduce:

```bash
cd backend && source .venv/bin/activate
python - <<'PY'
# (script used to produce the table — see task 34 notes)
PY
```

## Results (one representative populated row; bytes are UTF-8)

| Endpoint | DTO | Fields before | Fields after | Bytes/row before | Bytes/row after | Page@50 before | Page@50 after | Reduction |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| `GET /api/v1/vat/work-items` (and grouped/by-client) | `VatWorkItemResponse` → `VatWorkItemListItem` | 36 | 18 | 1094 | 471 | 54 700 | 23 550 | **56.9%** |
| `GET /api/v1/clients` | `ClientRecordResponse` → `ClientRecordListItem` | 27 | 10 | 803 | 273 | 40 150 | 13 650 | **66.0%** |
| `GET /api/v1/charges` | `ChargeResponse` → `ChargeListItem` | 23 | 15 | 581 | 391 | 29 050 | 19 550 | **32.7%** |
| `GET /api/v1/notifications` | `NotificationResponse` → `NotificationListItem` | 24 | 12 | 765 | 523 | 38 250 | 26 150 | **31.6%** |

Actual savings on real data will be **larger** than the byte figures suggest for
VAT and clients, because the dropped fields include the most variable-length
ones (override justifications, addresses, full message bodies on the detail
side) while the populated sample uses short Hebrew strings.

## Notes / caveats

- **Notifications** previously had **no** `GET /{id}` detail endpoint — the
  detail drawer reused the list row, which forced the list DTO to stay fat. As
  part of this task a real `GET /api/v1/notifications/{notification_id}`
  (returns full `NotificationResponse`, `404 NOTIFICATION.NOT_FOUND` when
  missing) was added and the drawer now fetches it. The thin
  `NotificationListItem` still carries `content_snapshot` / `subject_snapshot` /
  `business_name` because the **bell drawer** and **per-client tab** render a
  message preview inline in their list rows; only routing/delivery/debug fields
  (`channel`, `entity_*`, `binder_id`, `*_at`, `error_message`, `retry_count`,
  `triggered_by`) moved to detail-only. This caps the notifications reduction at
  ~32%.
- **Clients** list additionally **stops loading annual turnover per page**. The
  thin row does not expose turnover, so the per-page
  `VatClientSummaryRepository.get_annual_turnover_by_client_ids` lookup (and the
  now-meaningless `tax_year` query param on the list endpoint) were removed. This
  is a query-cost win on top of the payload win.
- **VAT** still computes detail-only deadline/name fields in the serializer even
  though `response_model` filters them out of the list. Follow-up (not done
  here, to keep the change small): skip `statutory_deadline` / `filed_by_name`
  computation on the list path.
