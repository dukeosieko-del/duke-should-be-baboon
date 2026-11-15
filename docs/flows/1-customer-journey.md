# 1. Customer journey — service entry to fulfilment and exit

**Ground truth:** `src/app/page.tsx`, `src/components/ServiceCatalogClient.tsx`,
`src/components/OrderForm.tsx`, `src/components/MpesaModal.tsx`,
`src/app/api/orders/route.ts`, `src/app/api/orders/anonymous/route.ts`,
`src/app/api/mpesa/callback/route.ts`, `src/lib/smm/fulfillment.ts`.

## 1.1 Landing (no session)
```
GET /
  └─ src/app/page.tsx → <ServiceCatalogClient/>
       └─ dynamic import of ServiceCatalog (bypasses standalone RSC chunking bug,
           commit ebOM_ZuAh3)
            └─ getServiceCatalogue() → GET /api/services/catalogue
                 → listJanjezServices(active, placement=undefined)
```
- Catalog is DB-driven (`janjez_services`), not static. The old static
  `ORDER_SERVICES` was removed from 13 page-clients in commit `0818978`.
- `KNOWN_PLATFORMS` (10 platforms) is the fallback set when the DB has no
  services for a platform — the UI still renders the platform card with count 0.

## 1.2 Browse → select
```
/services                          → platform listing (filters by show_catalogue)
/services/[platform]               → KNOWN_PLATFORMS substring matching
/services/[platform]/[subcategory] → by subcategory
/services/others                   → categorizeServices() fallback
```
- `matchPlatform()` maps aliases: `google-maps` → `google-maps-reviews`,
  `x-twitter` → `x`.

## 1.3 Order form (authenticated)
```
OrderForm (client)
  ├─ quantity, link, runs/interval (drip-feed)
  ├─ [guest checkbox] → POST /api/orders/anonymous
  └─ [signed in]     → POST /api/orders  { payment_method: "wallet" | "mpesa" }
```
- Price is shown client-side from the service row but **re-verified server-side**.

## 1.4 Payment fork

### Wallet path (already topped up)
```
POST /api/orders  payment_method="wallet"
  → debitWallet RPC
  → INSERT orders (payment_status="paid")
  → fulfillOrder(order.id)   ← synchronous
  → notification "Order Placed"
  → Brevo "Order Received" email
```

### M-Pesa path
```
POST /api/orders  payment_method="mpesa"
  → INSERT orders (payment_status="pending_mpesa")   ← NOT fulfilled yet
  → return { order, mpesa: { amount, order_id } }
  → client opens MpesaModal
  → POST /api/mpesa/stk-push  { orderId }
     → INSERT wallet_transactions (related_order_id, status="pending")
     → UPDATE orders payment_status="pending_mpesa"
     → initiateStkPush → Safaricom
  → client polls /api/mpesa/check-status
  → Safaricom callback → POST /api/mpesa/callback
     → completeStkPayment
        → UPDATE orders payment_status="paid", status="processing"
        → fulfillOrder(order.id)
```

### Guest path
```
POST /api/orders/anonymous
  → INSERT orders (user_id=null, anonymous=true, payment_status="pending_mpesa")
  → initiateStkPush (accountReference = order_id)
  → INSERT wallet_transactions (user_id=null, related_order_id)
  → callback → completeStkPayment → fulfillOrder
```

## 1.5 Fulfilment → exit
```
fulfillOrder
  → placeProviderOrder → provider_order_id
  → UPDATE orders fulfillment_status="processing"
  → (cron, every 5 min) syncOrderStatuses
     → provider "Completed" → fulfillment_status="fulfilled"
     → notification "Order ... status updated"
     → Brevo "Order Completed" email
```
- **Exit states:**
  - `fulfilled` → customer email + notification, order visible in `/orders/all`.
  - `failed` (after 3 retries) → refund via `credit_wallet`, `payment_status="refunded"`,
    Brevo "Order Failed — refund issued" email, admin notification.
  - `cancelled` (by admin) → `createProviderCancel`, status="cancelled".

## 1.6 Tracking
- **Public:** `/orders/track` — no auth, middleware explicitly allows it.
- **Authenticated:** `/orders/all` — middleware protects `/orders/:path*` except track.

## Rationale
- **The M-Pesa callback is the only place an M-Pesa order is fulfilled.** This
  ties money and fulfilment causally. If the callback is delayed or retried, the
  order stays `pending_mpesa` and is never double-fulfilled.
- **Guest orders have `user_id: null`.** They are reconciled purely through
  `wallet_transactions.related_order_id` and surfaced by `order_id`.
