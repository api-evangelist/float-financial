---
name: float-financial-spend-reporting
description: >-
  Build custom spend reports and dashboards from Float — pull card spend, account activity, bills,
  reimbursements and payments across a date window and attribute them to cards, users, teams, vendors and GL
  codes. The read-only reporting flow, with no writes.
api: Float Public API
base_url: https://api.floatfinancial.com
generated: '2026-08-16'
method: generated
source: >-
  openapi/float-financial-openapi.yml (operationIds verified against the spec),
  https://docs.floatfinancial.com/docs/accounting and
  https://help.floatfinancial.com/hc/en-us/articles/38048585600404-Get-Started-with-Float-s-Public-API
operations:
  - getTransactions
  - getAccountTransactions
  - getAccounts
  - getBills
  - getReimbursements
  - getPayments
  - getReceipts
  - getCards
  - getCardLimits
  - getUsers
  - getTeams
  - getVendors
  - getGLCodes
  - getTaxCodes
  - getSubsidiaries
  - getApprovalPolicies
  - getSubmissionPolicies
---

# Build a spend report from Float

Float names this as one of the three things its API is for: "Build custom reports — get real-time access to all
Float spend and deposit data." Everything here is read-only. No operation below mutates anything.

## Auth and shape

`Authorization: Bearer <token>`, one token per business, minted at app.floatfinancial.com → Settings →
Business Settings → Developers. No sandbox exists, but reads are safe.

Every collection operation takes the same four parameters:

```
?page=1&page_size=100&created_at__gte=<iso>&created_at__lte=<iso>&order_by=<field>
```

Page until `page.pages <= page_num`; rows are in `page.items`. Pin `created_at__lte` at the start of a run so
the window does not drift while you page.

## 1. Load the dimensions once

These are small, slow-changing, and every fact table references them. Cache them for the run.

- `getUsers`, `getTeams` — who spent, and under which team.
- `getCards` (+ `getCardLimits`) — the card, its user, currency, last four digits, spending power and limit.
- `getVendors`, `getGLCodes`, `getTaxCodes` — the coding dimensions.
- `getAccounts` — CAD/USD accounts and their types.
- `getSubsidiaries` — only if the business has a NetSuite connection; otherwise these return **403**
  (`Subsidiaries are only supported for Netsuite connections.`). Probe once and skip.
- `getApprovalPolicies`, `getSubmissionPolicies` — the control context, if you are reporting on policy
  compliance.

## 2. Pull the facts

- **`getTransactions`** — card spend, the primary fact table. Each row carries `total` and `merchant_total`
  (`merchant_total` is what the merchant charged; `total` is the settled amount, so FX shows up as the gap),
  `merchant`, `transacted_at` / `authorized_at`, `review_status`, `accounting_stage`,
  `spend_compliance_status`, and reference wrappers for card, user, team, vendor and subsidiary. Filter with
  `accounting_stage`, `transaction_type` or `card_id`.
- **`getAccountTransactions`** — funding, top-ups and cashback. These are account-level, not card-level, and are
  the deposit side of "spend and deposit data". Filter with `account_id`, `transaction_type` or `filter_type`.
- **`getBills`** — accounts payable, with `payee`, `total`, `status`, `export_status` and line items carrying
  their own GL and tax codes.
- **`getReimbursements`** — employee expense reports, with `requester`, `team`, `total`, `payment_status`.
- **`getPayments`** — the money movement behind bills and reimbursements. `resource_type` (a `PayableType`)
  discriminates whether `resource_id` points at a bill or a reimbursement; branch on it.
- **`getReceipts`** — receipt records with `image_url` and the transactions they match, for evidence links.

## 3. Attribute without extra calls

Float inlines related objects as reference wrappers rather than bare foreign keys, so most attribution needs no
join: `card` and `team` carry `{id, name}`, `user` carries `{id, email}`, `vendor` and `gl_code` carry
`{id, external_id}`, and `subsidiary` carries `{id, name, external_id}`.

There is no `expand` or `fields` parameter — you get the whole object, always. Budget for payload size rather
than for round trips.

`external_id` is the key to join Float against the ERP. Use it, not the Float UUID, when the report has to
reconcile to the general ledger.

## 4. Currency

Amounts are `MoneyValueSchema` objects, and the business can run both CAD and USD accounts and cards. Do not sum
across currencies — group by currency, or convert explicitly with a rate you control and record.

## 5. Practical limits

- **No published rate limits and no rate-limit headers.** Float's own guide shows an unbounded polling loop.
  Serialize the pages, add your own delay, and back off on any non-2xx.
- **No cursors.** Page-number paging over a moving collection can skip or repeat rows, which is exactly why you
  pin `created_at__lte`.
- **No aggregate endpoints.** There is no totals, summary or group-by operation — aggregation happens on your
  side, over the full pulled window.
- **No 429 or 5xx declared** in the spec. Handle them anyway.
- **Errors** arrive as `{"error", "message", "docs"}` in `application/json`, undeclared in the OpenAPI. Branch
  on `error`.

## Lower-effort alternative

For a one-off report with no code, Float publishes a Google Sheets template: copy it, paste an API token and
business name, and click "Import data".
