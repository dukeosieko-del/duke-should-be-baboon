# 6. Business Side API (the "dripfeeds" island)

**Ground truth:** `src/lib/business-side/{config,auth,orders,wallet,account,services,response,rate-limit,idempotency,types}.ts`,
`src/app/api/business/v1/**`.

## What it is
A separate HMAC-authenticated API island that lets an external "Business Side"
system (the `ez-business-side` package) place orders, check balances, and top up
its wallet **without** going through the customer website. It is **not** a
"dripfeeds service" in the sense of a drip-feed provider — "dripfeed" here is
the *DripFeedPanel SMM provider* that the whole system uses; the Business Side
is the *caller*.

## Auth — HMAC request signing (`authenticateBusinessSideRequest`)
Every request must carry:
```
x-business-side-api-key   : <API key>          (constant-time compare)
x-timestamp               : epoch seconds       (|now - ts| <= 300s)
x-nonce                   : unique string       (200 char max, 10-min TTL, replay-guarded)
x-signature               : 64-hex-char HMAC-SHA256
```
Signature is computed over:
```
METHOD\nPATH\nHMAC-SHA256(rawBody)\nTIMESTAMP\nNONCE
```
- `x-business-side-api-key` from `BUSINESS_SIDE_API_KEY` env.
- HMAC secret from `BUSINESS_SIDE_HMAC_SECRET || JANJEZ_MAIN_API_SECRET`.
- Nonces stored in an in-memory `Map<nonce, expiresAt>`, pruned on each call.

## Rate limit (`businessRateLimit`)
60-second sliding window, default 120 req/min, keyed by `ip:apiKey`.

## Response envelope (`businessSuccess` / `businessError`)
```json
{ "success": true,  "data": ..., "error": null,  "request_id": "<uuid>", "timestamp": "<iso>" }
{ "success": false, "data": null, "error": { "code", "message" }, "request_id": "<uuid>", "timestamp": "<iso>" }
```

## Routes

| Route | Method | Auth | Purpose |
|---|---|---|---|
| `/api/business/v1/health` | GET | HMAC | liveness |
| `/api/business/v1/services` | GET | HMAC | paginated service list (`toBusinessService` strips `provider_service_id`) |
| `/api/business/v1/services/[id]` | GET | HMAC | single service |
| `/api/business/v1/orders` | POST | HMAC + `Idempotency-Key` header | create order |
| `/api/business/v1/orders/[id]` | GET | HMAC | read order |
| `/api/business/v1/orders/[id]/cancel` | POST | HMAC | cancel |
| `/api/business/v1/orders/[id]/refill` | POST | HMAC | refill |
| `/api/business/v1/wallet/balance` | GET | HMAC | balance |
| `/api/business/v1/wallet/topup` | POST | HMAC + `Idempotency-Key` | M-Pesa STK top-up |
| `/api/business/v1/oauth/token` | POST | HMAC | exchange OAuth code → user_id/email (JWT-verified with `SUPABASE_SERVICE_ROLE_KEY`, `client_id === 'ez-business-side'`) |
| `/api/business/v1/webhooks/register` | POST | HMAC | webhook registration |

## Order creation (`createBusinessOrder`)
```
1. resolve service via resolveJanjezService(skuId, catalogCategoryId, janjezServiceId)
2. validate quantity within [min_quantity, max_quantity]
3. IDEMPOTENCY: SELECT orders WHERE user_id=accountId AND payment_reference="business-side:<key>"
   → replayed: true if found
4. amount = calculateOrderCost(selling_price_ksh, quantity)
5. debit_wallet(accountId, amount)   ← must succeed
6. INSERT orders { user_id: accountId, payment_status: "paid", payment_method: "wallet",
   payment_reference: "business-side:<key>", quantity_source: "preset", runs, interval }
7. fulfillOrder(order.id)   ← synchronous
8. on fulfillment error: credit_wallet (refund) + payment_status="refunded"
```
- The Business Side account is a **normal Supabase profile** identified by
  `BUSINESS_SIDE_ACCOUNT_ID`. It pays from its own `wallet_balance`.
- `provider_service_id` is stripped from the response (`safeOrder` destructuring
  in the route + `toBusinessService`).

## Wallet top-up (`topUpBusinessWallet`)
```
1. validate account exists
2. IDEMPOTENCY: SELECT wallet_transactions WHERE user_id=accountId AND
   notes="business-side:<key>" → replayed: true if found
3. initiateStkPush({ phoneNumber, amount, callbackUrl, accountReference:
   "business-side:<key>", transactionDesc: "Business Side wallet top-up" })
4. INSERT wallet_transactions { user_id: accountId, type: "topup", amount,
   payment_method: "mpesa", reference: CheckoutRequestID, status: "pending",
   notes: "business-side:<key>" }
```
- The M-Pesa callback (B2) settles it like any other top-up.

## Idempotency (`withIdempotency`)
In-memory `Map<key, {expiresAt, value}>` + `inFlight` promise dedup. TTL 24h.
`clearIdempotencyForTests()` for tests.

## Rationale
- **HMAC + nonce + timestamp + idempotency-key is defense in depth.** API key
  authenticates, HMAC proves the request was made by the holder of the secret,
  timestamp prevents replay after 5 min, nonce prevents same-second replay, and
  `Idempotency-Key` makes order creation safely retryable.
- **The Business Side is a first-class wallet holder**, not a special role. It
  uses the same `debit_wallet`/`credit_wallet` RPCs and the same
  `fulfillOrder` as a human customer. The only difference is the auth layer.
- **`provider_service_id` is stripped twice** (route destructuring +
  `toBusinessService`) because it is an internal DripFeedPanel reference that
  must never leak to a third-party integrator.