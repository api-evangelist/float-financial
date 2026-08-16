---
name: float-financial-accounting-sync
description: >-
  Sync Float card transactions, bills and reimbursements into an external accounting system (or a data
  warehouse), and mark them exported. This is the flow Float's own accounting guide is written around.
api: Float Public API
base_url: https://api.floatfinancial.com
generated: '2026-08-16'
method: generated
source: >-
  openapi/float-financial-openapi.yml (operationIds verified against the spec) and
  https://docs.floatfinancial.com/docs/accounting
operations:
  - createAccountingConnection
  - getAccountingConnections
  - getGLCodes
  - createGLCodes
  - updateGLCode
  - getTaxCodes
  - createTaxCodes
  - tax_components_retrieve
  - getVendors
  - createVendors
  - patchVendor
  - getCustomFields
  - createCustomFields
  - getTransactions
  - getTransactionById
  - patchTransaction
  - patchTransactions
  - getBills
  - patchBill
  - markBillAsSynced
  - getReimbursements
  - patchReimbursement
  - markReimbursementAsSynced
  - getPayments
  - getAccountTransactions
---

# Sync Float to an accounting system

## Before you start

- **Auth**: every call carries `Authorization: Bearer <token>`. Tokens are minted per business by an
  Administrator at app.floatfinancial.com → Settings → Business Settings → Developers.
- **There is no sandbox.** Float states plainly that it offers no sandbox or test environment. Everything below
  runs against live business data. Do reads first; gate every write behind an explicit confirmation.
- **Do not run this if the business already has a packaged integration.** If Float is connected to NetSuite,
  QuickBooks Online or Xero, Float already syncs GL codes, vendors, custom fields and tax codes itself, and
  users export transactions from the web app. Running an API sync on top of a packaged connector will fight it.
  Check `getAccountingConnections` first.

## 1. Register the connection

```
POST /v1/accounting-connections
Authorization: Bearer <token>
X-Idempotency-Key: <uuid you generate>
Content-Type: application/json

{"provider_name": "my_provider"}
```

`createAccountingConnection` **requires** `X-Idempotency-Key` — generate a fresh UUID per logical attempt and
reuse the SAME key on retry. Success is **HTTP 202**, not 201. Confirm with `getAccountingConnections`.

## 2. Push the coding dimensions

Four object types drive how spend is coded. Read what Float already has, then create only what is missing —
match on `external_id`, which is the join key to the ERP.

| Read | Create | Update |
|---|---|---|
| `getGLCodes` | `createGLCodes` (bulk) | `updateGLCode` (requires `X-Idempotency-Key`) |
| `getTaxCodes` | `createTaxCodes` (bulk) | `updateTaxCode` |
| `getVendors` | `createVendors` (bulk) | `patchVendor` |
| `getCustomFields` | `createCustomFields` (bulk) | `updateCustomField` (requires `X-Idempotency-Key`) |

- GL codes form a hierarchy through `parent_external_id`, keyed on `external_id` — create parents first.
- Multi-part Canadian tax codes (GST/PST/HST) compose components; list what is available with
  `tax_components_retrieve` before calling `createTaxCodes`.
- `updateTaxCode` **replaces all existing components** when `components` is supplied. Send the full set.
- The bulk create operations do **not** accept `X-Idempotency-Key`. A retried bulk create can duplicate rows —
  re-read with the matching `get*` operation and diff on `external_id` instead of blind-retrying.

## 3. Pull spend

All collection operations page identically:

```
GET /v1/card-transactions?page=1&page_size=100&created_at__gte=<iso>&created_at__lte=<iso>
```

Loop until `page.pages <= page_num`; rows are in `page.items`. `created_at__gte` / `created_at__lte` and
`order_by` are available on all 21 collection operations — use a closed `created_at` window so a long-running
sync is not disturbed by rows arriving mid-run.

- `getTransactions` — card transactions. Filter with `accounting_stage` to take only what is ready.
- `getBills` — accounts payable. `getBillAttachments` / `getBillAttachmentById` for invoice files.
- `getReimbursements` — employee expense reports.
- `getPayments` — payments against bills and reimbursements. `resource_type` (a `PayableType`) tells you whether
  `resource_id` is a bill or a reimbursement; branch on it before resolving the parent.
- `getAccountTransactions` — funding, top-ups and cashback, which are account-level, not card-level.

Float publishes **no rate limits and returns no rate-limit headers**. Its own guide shows an unbounded polling
loop. Pace the loop yourself and back off on any non-2xx.

## 4. Code and write back

- One transaction: `patchTransaction` on `/v1/card-transactions/{transaction_id}`.
- Many: `patchTransactions` on `/v1/card-transactions`.
- Bills: `patchBill` / `patchBills`. Reimbursements: `patchReimbursement` / `patchReimbursements`.

None of these PATCH operations accepts `X-Idempotency-Key`. Make them safe by re-reading the object and
comparing before writing, not by retrying.

## 5. Mark as exported

- `markBillAsSynced` → `POST /v1/bills/{bill_id}/sync`
- `markReimbursementAsSynced` → `POST /v1/reimbursements/{reimbursement_id}/sync`

Both **require** `X-Idempotency-Key`. Only call them after the accounting system has confirmed the write —
this is the transition that tells Float's users the object left the system.

Card transactions have no `sync` operation; move them by setting `accounting_stage` through
`patchTransaction`.

## 6. Reconcile

Re-read the window with `created_at__lte` pinned to the same upper bound and compare `export_status` /
`accounting_stage` against your own ledger. Anything still unexported is the retry set.

## Errors

Float returns `{"error": "...", "message": "...", "docs": "..."}` with `application/json` — **not** RFC 9457,
and **not declared in the OpenAPI**, so a generated client will have no error type. Branch on `error`.

| Status | Meaning | Do |
|---|---|---|
| 400 | Invalid request data | Fix the body against the operation's requestBody schema. |
| 401 | Bad or missing token | Reissue from the Float web app. Not declared in the spec, but returned live. |
| 403 | `Subsidiaries are only supported for Netsuite connections.` | Skip the subsidiary endpoints unless the business is on NetSuite. |
| 404 | Not found | Also what you get for an object outside your token's business. Verify the UUID. |
| 422 | e.g. `start_date is not enabled for this business` | Feature-gated field — omit it or ask Float to enable it. |
| 429 | Undeclared and undocumented | Treat defensively: exponential backoff. |
