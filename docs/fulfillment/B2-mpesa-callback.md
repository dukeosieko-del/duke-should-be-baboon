# B2 — M-Pesa Callback (Safaricom → Janjez)

**Ground truth:** `src/app/api/mpesa/callback/route.ts:36` (`POST`),
`src/lib/mpesa/client.ts:195` (`completeStkPayment`).

## Entry
Safaricom POSTs to `CallBackURL` = `getCallbackUrl(SITE_URL)` =
`https://janjez.social/api/mpesa/callback` (or `MPESA_CALLBACK_URL` env override).

## Flow
```
POST /api/mpesa/callback  { CheckoutRequestID, ResultCode, ResultDesc, CallbackMetadata }
  │
  ├─ rateLimit(30)
  ├─ createAdminClient()
  ├─ checkoutRequestID = body.CheckoutRequestID
  │   └─ missing → 400
  │
  ├─ resultCode = body.ResultCode || "999"
  │
  ├─ [resultCode !== "0"]   # FAILURE path
  │   └─ UPDATE wallet_transactions
  │        SET status="failed", notes=ResultDesc || ResponseDescription
  │        WHERE reference=checkoutRequestID AND status="pending"
  │   └─ return { ok: true, resultCode, credited: false }
  │
  └─ [resultCode === "0"]   # SUCCESS path
       ├─ metadata = extractMetadata(body)
       │   { amount, phone, receipt: ReceiptNumber || MpesaReceiptNumber }
       │
       ├─ completeStkPayment(checkoutRequestID, metadata)
       │
       │   completeStkPayment (src/lib/mpesa/client.ts:195):
       │     ├─ SELECT id,user_id,amount,related_order_id,status
       │     │   FROM wallet_transactions WHERE reference=checkoutRequestID
       │     │   └─ not found → throw "No transaction found"
       │     │
       │     ├─ amount = pendingTx.amount (requested amount is authoritative)
       │     │
       │     ├─ [status === "completed"]   # IDEMPOTENCY — Safaricom retries
       │     │   └─ return { newBalance, amount, orderId, alreadyCompleted: true }
       │     │
       │     ├─ [status !== "pending"] → throw "Transaction is already <status>"
       │     │
       │     ├─ UPDATE wallet_transactions
       │     │   SET status="completed", mpesa_receipt, mpesa_phone, notes
       │     │   WHERE reference AND status="pending"
       │     │
       │     ├─ [related_order_id present]   # ORDER PAYMENT
       │     │   ├─ UPDATE orders SET payment_status="paid", status="processing",
       │     │   │   fulfillment_status="pending"
       │     │   │   WHERE id AND payment_status IN ("pending_mpesa","pending")
       │     │   ├─ await fulfillOrder(related_order_id)   # ← triggers SMM fulfillment
       │     │   ├─ INSERT notification (type: "topup", "Payment Received")
       │     │   └─ return { newBalance, amount, orderId }
       │     │
       │     └─ [no related_order_id]   # WALLET TOP-UP
       │         ├─ [no user_id] → log, return { newBalance: 0, amount }
       │         └─ creditWallet(supabase, user_id, amount)   # RPC credit_wallet
       │            ├─ INSERT notification "Wallet Top-Up Successful"
       │            └─ return { newBalance, amount }
       │
       ├─ [result.amount > 0]  → fire-and-forget Brevo "Payment Received" email
       │   (lookup user by tx.user_id → auth.admin.getUserById → email)
       │
       └─ return { ok: true, resultCode, resultType, credited: true }
```
**On any throw inside `completeStkPayment`:** the callback returns **500** and
*leaves the transaction PENDING* (does not mark it failed), so a reconciliation
job can retry the credit instead of losing the deposit. Safaricom redelivers on
500.

## Exit
`{ ok: true, credited: true }` — money is in the user's wallet or the order is
marked paid and fulfillment has been triggered.

## Rationale
- **Idempotency is the whole point.** Safaricom retries callbacks. The
  `status="completed"` early-return and the `WHERE status="pending"` on the
  update prevent double-credit.
- **The requested amount is authoritative**, not the callback's `Amount`.
  A mismatch is logged but the requested amount is used — the customer cannot
  be charged more than they authorized.
- **Order payment ≠ wallet top-up.** When `related_order_id` is set, the money
  is *not* credited to the wallet; the order is marked paid and fulfillment is
  triggered. This is the critical branch split.
- **A failed callback never marks the transaction failed silently** — it sets
  `status="failed"` so the admin ledger shows it, but the 500 path preserves
  the pending state for retry.