---
name: plateiq-sync-erp-master-data
description: Bulk-sync vendors and accounting dimensions from an ERP into Ottimate and verify the results, so invoices can be coded and matched.
api: Ottimate API
base_url: https://api.ottimate.com/v1
operations:
  - post-vendors-bulk
  - get-vendors-root
  - get-vendors-id
  - post-dimensions-bulk
  - get-dimensions-root
  - get-batch-id-progress
  - get-batch-id-results
generated: '2026-07-20'
method: generated
source: openapi/plateiq-openapi.json
---

# Sync ERP master data into Ottimate

Keeps Ottimate's vendors and accounting dimensions in step with the source ERP so
invoices map cleanly to the chart of accounts.

## Auth (every request)
Send BOTH `X-API-Key` and `Authorization: Bearer <token>`. Add `Idempotency-Key`
on every bulk write.

## Steps
1. **Bulk upsert vendors** — `post-vendors-bulk` (`POST /vendors/bulk`) with the vendor list keyed by `erp_vendor_id`. Large payloads process asynchronously and return a batch id.
2. **Bulk upsert dimensions** — `post-dimensions-bulk` (`POST /dimensions/bulk`) for cost centers, departments, classes, projects, and custom attributes keyed by `erp_dimension_id`.
3. **Track async batches** — for either bulk call, poll `get-batch-id-progress` (`GET /batch/{id}/progress`) until complete, then `get-batch-id-results` (`GET /batch/{id}/results`) to read created/updated/error items.
4. **Verify** — `get-vendors-root` (`GET /vendors`) and `get-dimensions-root` (`GET /dimensions`) with `search`/paging (`page`, `size`) to confirm the records landed; `get-vendors-id` (`GET /vendors/{id}`) to spot-check one.

## Rules
- Dimensions must exist in Ottimate before they can be referenced on PO/invoice line items.
- Idempotency scope is `API key + method + path + key`; reuse the same key to safely retry a failed bulk load with the identical body.
- Respect the 10,000 req/day quota — prefer bulk endpoints over per-record calls for initial loads.
