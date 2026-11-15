# C2 — Anonymous (guest) checkout

**Ground truth:** `src/app/api/orders/anonymous/route.ts:23` (`POST`).

## Entry
`OrderForm.tsx` with "Place order as guest (no account needed)" checked →
`POST /api/orders/anonymous`.

## Flow
```
POST /api/orders/anonymous { janjez_service_id, link_submitted, quantity,
                             phone_number, runs?, interval?, email? }
  │
  ├─ rateLimit(20)
  ├─ validation: janjez_service_id, phone (9-15 digits), optional email regex,
  │   quantity≥1, valid link, runs/interval
  │
  ├─ listJanjezServices(true, "show_anonymous")   # only show_anonymous=true services
  ├─ find service by id
  │   ├─ not found → 404
  │   ├─ no provider_service_id → 400 "not configured for ordering"
  │   └─ quantity outside [min,max] → 400
  │
  ├─ expectedAmount = calculateOrderCost(selling_price_ksh, quantity)
  ├─ orderId = `ORD-${Date.now().toString(36).toUpperCase()}`
  ├─ amountForMpesa = calculateMpesaAmount(expectedAmount)
  │
  ├─ INSERT orders { order_id: orderId, user_id: null, phone_number, anonymous: true,
  │   category, subcategory, service_name, sku_id, janjez_service_id,
  │   link_submitted, link, quantity, amount: expectedAmount,
  │   amount_paid: amountForMpesa, quantity_source: "preset", status: "pending",
  │   payment_status: "pending_mpesa", fulfillment_status: "pending",
  │   runs, interval }
  │
  ├─ [email provided] → fire-and-forget Brevo "Order Received" email
  │
  ├─ initiateStkPush({ phoneNumber, amount: amountForMpesa,
  │   callbackUrl: getCallbackUrl(SITE_URL),
  │   accountReference: orderId,
  │   transactionDesc: "Anonymous order payment for <service>" })
  │
  ├─ INSERT wallet_transactions { user_id: null, phone_number, type: "order_payment",
  │   amount: amountForMpesa, payment_method: "mpesa", mpesa_phone,
  │   reference: CheckoutRequestID, related_order_id: order.id, status: "pending",
  │   notes: "Anonymous order STK for <orderId>" }
  │
  └─ return { ok, order_id, checkoutRequestId, amount, amountForMpesa, message }
```
**On STK error:** UPDATE orders fulfillment_status="failed",
fulfillment_error=..., return `{ ok: false, order_id, error }` with status 200
(so the client can show a friendly message without an HTTP error).

## Exit
`{ ok: true, order_id, checkoutRequestId, ... }`. The M-Pesa callback (B2)
settles it: `completeStkPayment` sees `related_order_id`, marks the order paid,
and calls `fulfillOrder`.

## Rationale
- **`user_id: null`.** There is no account. The order is keyed by `order_id`
  and the callback reconciles via `wallet_transactions.related_order_id`.
- **`amount_paid` ≠ `amount`.** `amount` is the service price; `amount_paid`
  includes the KES 7 service charge that Safaricom collects. The callback uses
  `pendingTx.amount` (the requested amount) as authoritative.
- **`show_anonymous` placement flag gates this route.** A service must opt in
  to guest checkout (`src/lib/janjez-services.ts` `createJanjezService` default
  `show_anonymous: true`).
- **Email is optional.** A guest can pay without revealing an email; if they do
  provide one they get a receipt.