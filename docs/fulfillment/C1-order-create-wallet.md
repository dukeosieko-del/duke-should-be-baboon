# C1 — Order Creation (authenticated user, wallet)

**Ground truth:** `src/app/api/orders/route.ts:124` (`POST`).

## Entry
`OrderForm.tsx` → `POST /api/orders` with `payment_method: "wallet"` (default).

## Flow
```
POST /api/orders { category, subcategory, sku_id, quantity, link_submitted,
                   amount_paid, catalog_category_id, payment_reference?,
                   refill_guarantee?, quantity_source, runs?, interval?,
                   janjez_service_id?, payment_method? }
  │
  ├─ rateLimit(30)
  ├─ getUserFromRequest → 401 if none
  ├─ validation: category, subcategory (unless janjez_service_id), quantity≥1,
  │   valid link, amount_paid≥0, runs/interval integers >0
  │
  ├─ resolve service:
  │   if janjez_service_id → resolveJanjezService(null,null,id)
  │     ├─ not found → 404
  │     ├─ quantity outside [min,max] → 400
  │     └─ runs/interval set but !supports_drip_feed → 400
  │   else → calculateExpectedAmount(catalog_category_id, category, subcategory, sku_id, quantity)
  │     (parses static SERVICE_CATALOG price strings — legacy fallback)
  │
  ├─ price check: |expectedAmount - clientAmount| > 0.01 → 400 "Price verification failed"
  │
  ├─ paymentMethod = "wallet" | "mpesa"
  │
  ├─ [wallet]
  │   └─ debitWallet(supabase, user.id, expectedAmount)   # RPC debit_wallet
  │       ├─ !success → 402 "Insufficient wallet balance"
  │       │   └─ INSERT notification "Order Failed - Insufficient Wallet Balance"
  │       └─ newBalance = debit.newBalance
  │
  ├─ fireAndForgetEmail(maybeSendLowBalanceEmail(...))   # if balance < KES 100, 24h debounce
  │
  ├─ INSERT orders { user_id, order_id?, category, subcategory, service_name,
  │   sku_id, quantity, link_submitted, link, amount, amount_paid,
  │   payment_reference, refill_guarantee, quantity_source, catalog_category_id,
  │   janjez_service_id, status:"pending", payment_status:"paid"|"pending_mpesa",
  │   fulfillment_status:"pending", runs, interval }
  │
  ├─ [mpesa] → return early { order, mpesa: { amount, order_id } }
  │             (the M-Pesa callback settles it — do NOT fulfill here)
  │
  └─ [wallet] → fulfillOrder(data.id)
        ├─ "processing"|"already_fulfilled" → INSERT notification "Order Accepted"
        ├─ error → UPDATE orders fulfillment_status="failed", fulfillment_error,
        │   INSERT notification "Order Fulfillment Failed",
        │   creditWallet(user.id, expectedAmount)   # REFUND
        │   fireAndForgetEmail(getOrderFailedEmail(...))
        └─ INSERT notification "Order Placed"
           fireAndForgetEmail(getOrderReceivedEmail(...))
```

## Exit
`{ order }` 201. Order is being fulfilled.

## Rationale
- **Price is server-authoritative.** The client submits `amount_paid` but the
  server re-derives `expectedAmount` from the service row and rejects mismatches.
  This is the single most important fraud control in the build.
- **M-Pesa orders are NOT fulfilled in this route.** They are fulfilled inside
  `completeStkPayment` (B2) so the money and the fulfilment are causally tied.
  Fulfilling here would create orders that are paid-for but never fulfilled if
  the M-Pesa flow is abandoned.
- **Refund on fulfillment failure.** If the provider rejects the order after
  the wallet was debited, the money is credited back immediately.