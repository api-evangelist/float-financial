---
name: float-financial-webhook-consumer
description: >-
  Register a Float webhook endpoint, verify the HMAC-SHA256 signature on every delivery, and hydrate the thin
  event payload into a full card transaction. Covers all four card-transaction events Float emits.
api: Float Public API
base_url: https://api.floatfinancial.com
generated: '2026-08-16'
method: generated
source: >-
  https://docs.floatfinancial.com/docs/webhooks and openapi/float-financial-openapi.yml (operationIds verified
  against the spec)
operations:
  - createWebhookSubscription
  - getWebhookSubscriptions
  - getTransactionById
  - patchTransaction
---

# Consume Float webhooks

## What Float emits

Four events, all on card transactions. Nothing else in Float has webhook coverage — bills, reimbursements,
payments, receipts, cards, users and the coding objects must be polled.

| Event | Fires when |
|---|---|
| `transaction.authorized` | A card authorization is approved, in real time at purchase. |
| `transaction.cleared` | The transaction is captured and settled by the card network. |
| `transaction.ready_to_export` | The transaction reaches the `READY_TO_EXPORT` accounting stage after review. |
| `transaction.export_requested` | An export is requested for the transaction on the Month End page. |

## 1. Register the endpoint

```
POST /v1/webhooks
Authorization: Bearer <token>
Content-Type: application/json

{"url": "https://your-host.example.com/float/webhook"}
```

`createWebhookSubscription`. **The signing secret is returned once, at creation time only.** Persist it to your
secret store in the same code path that makes this call — there is no operation to read it back and no
operation to rotate or delete a subscription (`/v1/webhooks` exposes only GET and POST). Confirm the
registration with `getWebhookSubscriptions`.

There is no per-subscription event filter: you receive all four types and must dispatch on `type` yourself.

## 2. Verify every delivery

Each request carries three headers:

```
Float-Signature: sha256=<hex>
Float-Webhook-Id: <event id>
Float-Timestamp: <unix seconds>
```

Compute HMAC-SHA256 with the stored signing secret and compare against `Float-Signature` using a
constant-time comparison. Reject anything that fails.

Two things Float does **not** publish, which you must decide yourself:

- **Exactly what string is signed.** The docs say "the signed content" without defining the construction. Confirm
  it with Float support before going live rather than guessing — this is the single most error-prone step.
- **A timestamp tolerance.** `Float-Timestamp` is provided for replay protection but no window is specified.
  Reject deliveries older than your own threshold (five minutes is the usual choice).

## 3. Deduplicate

Float retries up to **10 times** with exponential backoff and jitter, treating HTTP **200–299** as success and
timing out at **10 seconds**. You will receive duplicates.

Use the event `id` (identical to `Float-Webhook-Id`) as the idempotency key. Record it before doing any work
and drop repeats. Ordering is not documented — do not assume `authorized` arrives before `cleared`.

## 4. Respond fast, then hydrate

Acknowledge with a 2xx inside the 10-second timeout, then do the work asynchronously.

The payload is deliberately thin:

```json
{
  "id": "evt_...",
  "type": "transaction.authorized",
  "created_at": "2026-08-16T14:00:00Z",
  "business_id": "<uuid>",
  "object": { "id": "<transaction uuid>" }
}
```

`object.id` is all you get. Hydrate with `getTransactionById`:

```
GET /v1/card-transactions/{transaction_id}
Authorization: Bearer <token>
```

That returns the full `CardTransactionSchema` — merchant, `total` and `merchant_total`, the card, user and team
references, receipts, lines, `accounting_stage`, `review_status` and `spend_compliance_status`.

## 5. Act on it

- On `transaction.ready_to_export` or `transaction.export_requested`, push the transaction to the accounting
  system, then advance `accounting_stage` with `patchTransaction`.
- `patchTransaction` does **not** accept `X-Idempotency-Key`. Re-read and compare before writing rather than
  retrying blind.

## Failure handling

Errors come back as `{"error": "...", "message": "...", "docs": "..."}`, not RFC 9457, and are not declared in
the OpenAPI. A 404 on `getTransactionById` after an event means the UUID is not visible to your token's
business — check you are using the token for the `business_id` in the event, not a different one.

Float publishes no rate limits and returns no rate-limit headers. A burst of authorizations produces a burst of
hydration calls: queue them and pace yourself.
