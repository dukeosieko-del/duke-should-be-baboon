# duke-should-be-baboon — document index

Surgical-repair documentation for the `janjez-socio` build (Next.js 16 + Supabase).

## Documents

### Flows
- `flows/0-iceberg-map.md` — the whole build in one view (layer map).
- `flows/1-customer-journey.md` — service entry → fulfilment → exit.
- `flows/2-admin-journey.md` — admin gate → service mgmt → ledger.
- `flows/13-walkthrough.md` — walkthrough engine.

### Auth
- `auth/A1-signup.md` — email/password signup + custom verification.
- `auth/A2-verify-email.md` — click link → confirm.
- `auth/A3-signin.md` — email/password sign-in + security alert.
- `auth/A4-google-oauth.md` — Google OAuth via Supabase.
- `auth/A5-password-reset.md` — request + set.
- `auth/A6-middleware-sessions.md` — 4 clients, middleware decision tree, admin gate.

### Fulfillment
- `fulfillment/B1-mpesa-stk-initiation.md` — STK push (wallet top-up + order).
- `fulfillment/B2-mpesa-callback.md` — Safaricom callback → credit/fulfil.
- `fulfillment/C1-order-create-wallet.md` — authenticated order (wallet).
- `fulfillment/C2-anonymous-checkout.md` — guest checkout.
- `fulfillment/D1-smm-provider.md` — DripFeedPanel provider layer.
- `fulfillment/D2-drip-feed-settings.md` — drip-feed limits.
- `fulfillment/E1-fulfillment-lifecycle.md` — fulfillOrder + retry + refund + cron sync.

### Business Side
- `business-side/6-business-side-api.md` — HMAC island (orders / wallet / oauth).

### Blog
- `blog/7-blog-system.md` — blog + comments + ratings + moderation.

### Infrastructure
- `infrastructure/3-brevo-email.md` — Brevo SMTP + sendTransactional.
- `infrastructure/4-supabase-provider.md` — the DB provider (PostgreSQL, no Prisma).
- `infrastructure/5-build-pipeline.md` — build / postbuild / CI / PM2.
- `infrastructure/10-security-invariants.md` — 12 security invariants.
- `infrastructure/11-deployment.md` — env vars, M-Pesa, DNS, targets.
- `infrastructure/12-prisma-migration-note.md` — explicit "no Prisma" finding.

### API
- `api/8-route-index.md` — every route with its auth model.

### Diagrams
- `diagrams/9-diagrams.md` — money flow, auth tree, fulfillment state machine,
  callback idempotency (ASCII + Mermaid).

### Figma
- `figma/index.md` — Figma tree → code route mapping.

## Ground-truth rule
Every document names the file path (and commit where relevant) that is its
authoritative source. When in doubt, the source file wins over this doc.