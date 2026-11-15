# 10. Security invariants

**Ground truth:** `src/middleware.ts`, `src/lib/server/auth-helpers.ts`,
`src/lib/supabase/admin.ts`, `src/lib/wallet.ts`, `src/lib/business-side/auth.ts`.

## I1 — Service-role key never reaches the browser
`createAdminClient()` (`src/lib/supabase/admin.ts`) reads `SUPABASE_SERVICE_ROLE_KEY`.
It is only imported from `src/app/api/**`, `src/lib/smm/**`,
`src/lib/business-side/**` — all server-only modules. The client bundle
(`src/lib/supabase/client.ts`) uses `NEXT_PUBLIC_SUPABASE_*`.

## I2 — Middleware is a redirect gate, not an authorization gate
`src/middleware.ts:48` lets admin pages *render* for any authenticated user; the
actual 403 comes from `requireAdmin()` in the API route. This prevents a forged
`user_metadata.role` from bypassing the API.

## I3 — Money moves only through atomic RPCs
All wallet debits/credits go through `debit_wallet` / `credit_wallet`
PL/pgSQL functions (`20250101000011_wallet_atomic_functions.sql`) which are
`BEGIN ... UPDATE ... IF NOT FOUND` transactions. `src/lib/wallet.ts` unwraps
the PostgREST array result (`data[0]`) — a historical bug read `data.new_balance`
directly and made failed debits look successful.

## I4 — Price is server-authoritative
`POST /api/orders` re-derives `expectedAmount` from the service row and rejects
any client-submitted `amount_paid` that differs by > 0.01. The client can never
decide how much it is charged.

## I5 — M-Pesa callback is idempotent
Safaricom retries callbacks. `completeStkPayment` early-returns on
`status === "completed"` and updates with `WHERE status = "pending"`, so a
replayed callback cannot double-credit.

## I6 — HMAC + nonce + timestamp + idempotency-key (Business Side)
`authenticateBusinessSideRequest` verifies API key (constant-time), HMAC-SHA256
over `METHOD\nPATH\nbodyHash\nTS\nNONCE`, timestamp within 300s, and nonce
uniqueness with a 10-min TTL. `Idempotency-Key` makes order creation safely
retryable.

## I7 — `provider_service_id` is stripped from customer/business responses
Two deliberate removals: `toBusinessService` (business-side API) and the
catalogue API projection. Commit `5adfbec`.

## I8 — Generic responses prevent user enumeration
`POST /api/auth/reset-password` returns the same message whether or not the
email exists.

## I9 — CSP + security headers on every response
`applySecurityHeaders` sets `Content-Security-Policy`, `X-Content-Type-Options`,
`Referrer-Policy`, `X-Frame-Options: DENY`, `Permissions-Policy`,
`Strict-Transport-Security`, `Cross-Origin-Opener-Policy`,
`Cross-Origin-Resource-Policy`, and `Cache-Control: no-store` (except static).

## I10 — Rate limiting on every write route
`rateLimit(request, N)` in `src/lib/server/rate-limiter.ts` guards signup (10),
reset (5), verify (60), orders (30), mpesa stk-push (20), callback (30),
check-status (30), security-alert (30). Business Side uses `businessRateLimit`
(120/min). Cron uses `rateLimitCron`.

## I11 — Brevo sender is verified
`BREVO_FROM_EMAIL` must be a verified Brevo sender; `BREVO_SMTP_USER`/`BREVO_SMTP_KEY`
are the SMTP login/key, deliberately distinct from the API key.

## I12 — Secrets never committed
`.env.local` is gitignored. `JANJEZ_BUILD_STATE.md` §G9: "Never record secrets
in the MD file." CI injects secrets via `${{ secrets.* }}`.