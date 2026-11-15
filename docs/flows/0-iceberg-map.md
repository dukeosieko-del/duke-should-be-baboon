# 0. Iceberg Map — the whole build in one view

```
                         ┌─────────────────────────────────────────────────────┐
                         │                     ENTRY POINTS                     │
                         │  /  /services  /blog  /auth/sign-in  /order/*      │
                         └──────┬──────────────┬──────────────┬────────────────┘
                                │              │              │
              ┌─────────────────┼──────────────┼──────────────┼──────────────┐
              ▼                 ▼              ▼              ▼              ▼
     ┌────────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
     │  PUBLIC CATALOG │  │  AUTH    │  │  BLOG    │  │  ORDER   │  │  PAY/WALLET│
     │  (no session)   │  │(session) │  │(session) │  │(session) │  │ (session)  │
     └───────┬─────────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬─────┘
             │                 │             │              │              │
             ▼                 ▼             ▼              ▼              ▼
     ┌────────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
     │ catalogue API  │  │  Supabase│  │ blog API │  │ orders   │  │ mpesa    │
     │ /api/services/ │  │  Auth    │  │ /api/blog│  │ /api/    │  │ /api/    │
     │ catalogue      │  │  +Google │  │          │  │ orders   │  │ mpesa    │
     └────────┬───────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬─────┘
              │               │             │              │              │
              └───────────────┼─────────────┼──────────────┼──────────────┘
                              │             │              │
                              ▼             ▼              ▼
                     ┌──────────────────────────────────────────────┐
                     │            DATA LAYER (Supabase PG)          │
                     │  profiles | orders | wallet_transactions        │
                     │  janjez_services | provider_services           │
                     │  blog_* | notifications | contact_messages      │
                     │  email_verifications | password_reset_tokens    │
                     │  fulfillment_logs | platform_settings         │
                     │  RPC: credit_wallet / debit_wallet           │
                     └──────────────────────┬───────────────────────┘
                                            │
                              ┌─────────────┴──────────────┐
                              ▼                            ▼
                     ┌──────────────────┐     ┌──────────────────────────┐
                     │  EXTERNAL        │     │  ADMIN + BUSINESS SIDE   │
                     │  PROVIDERS       │     │  /admin/* (role=admin)   │
                     │  • Safaricom     │     │  /api/business/v1/*      │
                     │    M-Pesa Daraja │     │    (HMAC + idempotency)   │
                     │  • DripFeedPanel │     │  cron: /api/cron/smm-sync│
                     │    SMM API v2    │     │                          │
                     │  • Brevo SMTP    │     │                          │
                     └──────────────────┘     └──────────────────────────┘
```

**Ground truth:** `src/middleware.ts`, `src/lib/supabase/{client,server,admin,middleware}.ts`,
`src/lib/business-side/*`, `src/lib/smm/fulfillment.ts`, `package.json`.

**Rationale for the layer split:** the customer surface (Next.js App Router
pages) is *never* the authority on money or on provider identity. Every write
path funnels through a server route that uses the **service-role admin client**
(`SUPABASE_SERVICE_ROLE_KEY`) and the atomic wallet RPCs. The customer surface
only ever reads via the anon client. This is the single most important security
invariant in the build — see `docs/infrastructure/security-invariants.md`.

**What is deliberately NOT here:**
- Prisma — does not exist (see README "DB provider note").
- A separate "dripfeed" microservice — DripFeed is the *SMM provider* name
  (`SMM_API_URL=https://dripfeedpanel.com/api/v2`). The "drip-feed" *feature*
  (scheduled repeated runs) is implemented inside `src/lib/smm/fulfillment.ts`
  and configured via `platform_settings.drip_feed_limits`.
- A standalone Google Auth service — Google OAuth is Supabase Auth's
  `signInWithOAuth({ provider: "google" })`, configured in the Supabase
  dashboard, not in code beyond the button + redirect.