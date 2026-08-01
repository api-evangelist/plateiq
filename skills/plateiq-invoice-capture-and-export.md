---
name: plateiq-invoice-capture-and-export
description: Capture a vendor invoice from a file, monitor extraction, review and code it, route approvals, and mark it exported using the Ottimate API.
api: Ottimate API
base_url: https://api.ottimate.com/v1
operations:
  - post-invoices-upload
  - get-invoices-uploads
  - get-invoices-root
  - get-invoices-id
  - patch-invoices-id
  - get-invoices-id-approvers
  - post-invoices-id-flag
  - post-invoices-mark-exported
generated: '2026-07-20'
method: generated
source: openapi/plateiq-openapi.json
---

# Capture and export an invoice with Ottimate

Automates Ottimate's InstantCapture -> Core AP flow: upload an invoice document,
let Ottimate extract the data, review/code it, confirm approvals, then export.

## Auth (every request)
Send BOTH headers on every call:
- `X-API-Key: <api_key>`
- `Authorization: Bearer <oauth2_access_token>` (client-credentials token from `/oauth/token`)

Send `Idempotency-Key: <uuid>` on every POST/PATCH write so retries never duplicate.

## Steps
1. **Upload the document** — `post-invoices-upload` (`POST /invoices/upload`) with the file or a download URL, plus `ottimate_company_id`/`ottimate_location_id` scope. Returns an upload/batch handle.
2. **Poll extraction** — `get-invoices-uploads` (`GET /invoices/uploads`) until the upload reports a completed status and an invoice id is available.
3. **Retrieve the invoice** — `get-invoices-id` (`GET /invoices/{id}`) to inspect the extracted header, line items, and current status.
4. **Code / correct it** — `patch-invoices-id` (`PATCH /invoices/{id}`) to set/fix dimensions on line items and header fields.
5. **Check approvers** — `get-invoices-id-approvers` (`GET /invoices/{id}/approvers`) to confirm the approval policy is satisfied. If something looks wrong, `post-invoices-id-flag` (`POST /invoices/{id}/flag`) to route for review.
6. **Export** — once approved, `post-invoices-mark-exported` (`POST /invoices/mark-exported`) with the invoice ids to mark them exported to the ERP.

## Rules
- Rate limits: 1,000 req/s steady, 2,000 burst, 10,000/day per API key; requests time out at 29s — treat uploads as async and poll.
- Errors use the `ErrorResponse` envelope (`code`, `message`, `request_id`, `timestamp`); cite `request_id` in support tickets.
- A `422` on a write with a reused `Idempotency-Key` means the body changed — use a new key for a genuinely new request.
