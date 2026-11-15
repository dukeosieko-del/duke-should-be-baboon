# E1 — Order fulfillment lifecycle

**Ground truth:** `src/lib/smm/fulfillment.ts:186` (`fulfillOrder`),
`src/app/api/orders/route.ts:331`, `src/app/api/mpesa/client.ts:287`,
`src/app/api/cron/smm-sync/route.ts:9`.

## The four entry points that call `fulfillOrder`

| Caller | Trigger | Context |
|---|---|---|
| `POST /api/orders` (wallet) | synchronous, in the route | user-paid order |
| `completeStkPayment` (M-Pesa callback) | synchronous, in the callback | M-Pesa-paid order |
| `createBusinessOrder` | synchronous, in the business-side API | business-side wallet order |
| `POST /api/cron/smm-sync` | **async, every 5 min** (GitHub Actions) | status sync only — does NOT call fulfillOrder |

## `fulfillOrder(orderId)` decision tree
```
SELECT * FROM orders WHERE id = orderId
  ├─ not found → throw
  ├─ provider_order_id IS NOT NULL → return { status: "already_fulfilled" }
  │
  ├─ SMM_FULFILLMENT_ENABLED !== "false"?  (env toggle)
  │   └─ false → UPDATE orders fulfillment_status="fulfillment_disabled", throw
  │
  ├─ retry_count >= MAX_RETRIES (3)?
  │   └─ true → logFulfillment "failed", UPDATE orders fulfillment_status="failed",
  │            fulfillment_error="Max retries exceeded", throw
  │
  ├─ resolve service:
  │   resolveJanjezService(sku_id, catalog_category_id, janjez_service_id)
  │   → SELECT provider_services WHERE id = service.provider_service_id
  │   → providerService = { service, name, ..., dripfeed }
  │   → quantity = clamp(order.quantity, provider.min, provider.max)
  │
  ├─ [no provider mapping] → logFulfillment "failed",
  │   UPDATE orders fulfillment_status="failed",
  │   fulfillment_error="No provider mapping found ...", throw
  │
  ├─ expectedCharge = (providerRate * quantity) / 1000
  │
  ├─ getProviderBalance() < expectedCharge?
  │   └─ true → logFulfillment "failed",
  │            UPDATE orders fulfillment_status="failed",
  │            fulfillment_error="Insufficient provider balance ...", throw
  │
  ├─ placeProviderOrder({ service, link, quantity, runs?, interval? })
  │   ├─ error / no order id → handleFulfillmentFailure(...)
  │   └─ success (response.order) ▼
  │
  └─ UPDATE orders SET provider_service_id, provider_order_id,
       provider_status="pending", provider_charge=expectedCharge,
       provider_currency="USD", fulfillment_status="processing", fulfilled_at=now
     logFulfillment "processing"
     return { status: "processing", providerOrderId }
```

## `handleFulfillmentFailure` (retry + refund)
```
nextRetryCount = retryCount + 1
isRetryable = nextRetryCount < MAX_RETRIES
newStatus = isRetryable ? "retry_pending" : "failed"
delay = [60s, 300s, 900s][nextRetryCount-1]  (1min, 5min, 15min)
nextRetryAt = now + delay  (if retryable)

UPDATE orders SET fulfillment_status=newStatus, fulfillment_error,
               retry_count=nextRetryCount, next_retry_at=nextRetryAt

[!isRetryable] → refundOrder(supabase, order, errorMessage)
   credit_wallet(user_id, amount_paid)
   UPDATE orders SET payment_status="refunded"

notifyAdminsOfFailure(...)   # in-app admin notification
```

## Status sync (`syncOrderStatuses`, cron)
```
SELECT orders WHERE provider_order_id IS NOT NULL
  AND (fulfillment_status IN ("pending","processing") OR orderIds given)

getProviderMultipleStatus(all provider_order_ids)

for each order:
  mapped = mapProviderStatus(status.status)
    "Completed"  → "fulfilled"
    "Cancelled"/"Refunded" → "cancelled"
    "Partial"    → "processing"
    "Pending"/"In Progress"/"Processing" → "processing"
    unknown      → "processing" + warn + logFulfillment "unknown_status"

  UPDATE orders SET provider_status, provider_start_count, provider_remains,
                  provider_charge, provider_currency, fulfillment_status=mapped

  [status changed] → INSERT notification "Order ... status updated"
  [failed/cancelled] → notifyAdminsOfFailure (admin notification)

  fireAndForgetEmail(maybeSendOrderStatusEmail(...))
    "fulfilled"  → getOrderCompletedEmail
    "failed"/"cancelled" → getOrderFailedEmail
```

## Refill / cancel (`requestRefill`, `requestCancel`, `refillOrder`, `cancelOrder`)
- `cancelOrder`: requires `resolvedService.supports_cancel`, calls
  `createProviderCancel`, sets `fulfillment_status="cancelled"`, `status="cancelled"`.
- `refillOrder`: requires `resolvedService.supports_refill`, calls
  `createProviderRefill`, sets `provider_status="Pending"`.

## Rationale
- **MAX_RETRIES = 3 with exponential-ish backoff (60/300/900s).** Beyond that the
  order is failed and the customer refunded. This bounds the blast radius of a
  provider outage without silently retrying forever.
- **Provider balance is checked before ordering.** An order that would exceed
  the provider balance is failed immediately rather than rejected by the
  provider mid-flight.
- **`fulfillment_disabled` is an env toggle** (`SMM_FULFILLMENT_ENABLED`), so
  fulfillment can be paused without code changes during provider maintenance.
- **The cron job only syncs status; it never creates orders.** Creation is
  always synchronous to a user action so the caller can return the
  `provider_order_id` immediately.