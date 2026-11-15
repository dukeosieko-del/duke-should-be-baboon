# B1 — M-Pesa STK Push Initiation (wallet top-up)

**Ground truth:** `src/components/MpesaModal.tsx:67` (`handleTopUp`),
`src/app/api/mpesa/stk-push/route.ts:11` (`POST`), `src/lib/mpesa/client.ts:98`
(`initiateStkPush`).

## Entry
`MpesaModal` (no `orderId`) → "Pay with M-Pesa" → `POST /api/mpesa/stk-push`.

## Flow
```
MpesaModal.handleTopUp(phoneNumber, amount)
  │
  ├─ mpesaAmount = calculateMpesaAmount(amount)   # +KES 7 service charge
  │
  ├─ POST /api/mpesa/stk-push  { phoneNumber, amount: mpesaAmount, orderId? }
  │   headers: Authorization: Bearer <access_token>
  │
  │   route:
  │     ├─ rateLimit(20)
  │     ├─ getUserFromRequest → 401 if no user
  │     ├─ validate phone + amount > 0
  │     ├─ numAmount = calculateMpesaAmount(amount)   # server re-derives, client cannot decide
  │     ├─ if orderId:
  │     │   └─ SELECT order WHERE id=orderId AND user_id=me
  │     │       ├─ not found → 404
  │     │       ├─ already paid → 409
  │     │       └─ chargeAmount = calculateMpesaAmount(order.amount_paid)
  │     │       relatedOrderId = order.id
  │     │
  │     ├─ initiateStkPush({ phoneNumber, amount: chargeAmount,
  │     │     callbackUrl: getCallbackUrl(SITE_URL),
  │     │     accountReference: "janjez-order-<8hex>" | "janjez-topup-<8hex>",
  │     │     transactionDesc: "Order payment ..." | "Wallet top-up for <email>" })
  │     │
  │     │   initiateStkPush (src/lib/mpesa/client.ts):
  │     │     ├─ getAccessToken()  # cached OAuth token, refresh 60s before expiry
  │     │     │   └─ GET {BASE_URL}/oauth/v1/generate?grant_type=client_credentials
  │     │     │      Basic base64(MPESA_CONSUMER_KEY:MPESA_CONSUMER_SECRET)
  │     │     ├─ formatPhone()  → 254XXXXXXXXX
  │     │     ├─ timestamp = YYYYMMDDHHMMSS
  │     │     ├─ password = base64(shortcode + passkey + timestamp)
  │     │     └─ POST {BASE_URL}/mpesa/stkpush/v1/processrequest
  │     │          Authorization: Bearer <token>
  │     │          { BusinessShortCode, Password, Timestamp, TransactionType:
  │     │            "CustomerBuyGoodsOnline", Amount, PartyA: "1329148",
  │     │            PartyB: "1799086", PhoneNumber, CallBackURL,
  │     │            AccountReference, TransactionDesc, QueueTimeOutURL }
  │     │
  │     ├─ INSERT wallet_transactions {
  │     │     user_id, type: "topup"|"order_payment", amount: chargeAmount,
  │     │     currency: "KES", payment_method: "mpesa", mpesa_phone,
  │     │     reference: CheckoutRequestID, related_order_id,
  │     │     status: "pending", notes }
  │     │
  │     └─ if orderId: UPDATE orders SET payment_status="pending_mpesa"
  │            WHERE id AND payment_status != "paid"
  │
  └─ return { ok, checkoutRequestId, amount, orderId, message }
```

## Client-side polling
`MpesaModal` polls `GET /api/mpesa/check-status?checkoutRequestId=...` every 5s,
timeout 120s:
- `paid=false` → keep polling or timeout → "Payment timed out"
- `paid=true && credited=true` → `step="success"`, `refreshProfile()`
- `paid=true && credited=false` → keep polling; at 120s → `step="settling"`
- poll error → back to `step="input"` with error

## Exit
Modal shows success; user navigates to `/deposit/success?amount=&ref=&type=`.
Wallet balance updated via the callback.

## Rationale
- **`PartyA`/`PartyB` are hardcoded** (`1329148` / `1799086`) — these are the
  Safaricom sandbox shortcodes. They must be swapped for production values via
  env when `MPESA_ENV=production`. This is a known hardcoding, flagged in the
  build state as an environment-variable gap.
- **The client never decides the amount.** When `orderId` is present the route
  re-derives `chargeAmount` from the order row. The client's `amount` field is
  only authoritative for a pure wallet top-up.
- **`getCallbackUrl(origin?)`** accepts an explicit origin (added in
  MILESTONE 10, commit `911ae3a`) so staging vs production callbacks are
  correct instead of always using `SITE_URL`.