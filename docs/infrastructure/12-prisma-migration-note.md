# 12. Prisma / DB-provider migration note

## Current state (ground truth, 2026-11-15)
Prisma was **never** used in this build.
`grep -ri prisma` across `/home/ubuntu/janjez-socio-mobile-worktree`,
`/home/ubuntu/ez-business-side`, and `/home/ubuntu/janjez-business-worktree`
returns **zero** source matches (only unrelated `.tsbuildinfo` build artifacts).

The database layer has **always** been **Supabase (PostgreSQL)** via
`@supabase/supabase-js` + `@supabase/ssr` + 30 hand-written SQL migrations.

## The only "provider changes" that ever happened
1. **Email transport: ZeptoMail → Brevo SMTP** (commit `7d1fb3e`).
2. **Supabase project hostname** changed from
   `snkgkcdnmhqaejpqftxn.supabase.co` (now NXDOMAIN) to
   `rousjavuooduvicaobuv.supabase.co` — a manual `.env` reconstruction.
3. **No database engine or ORM swap.**

## ★ FUTURE DB DEPLOYMENT TARGET: PRISMA ★

**Decision (recorded 2026-11-15): all future database deployments must target Prisma.**

This is a forward-looking directive, not a description of past work. The Supabase
raw-SQL layer is the *current* production state; Prisma is the *approved future*
state. The migration is **not yet started** — this document is the spec for it.

### Why Prisma
- **Type safety at the query boundary.** The current code uses
  `data as unknown as JanjezService` casts in 6+ places (`janzez-services.ts:118`,
  `fulfillment.ts:130`, `blog/queries.ts:305`). Prisma eliminates these.
- **Migration versioning.** 30 hand-rolled SQL files with `IF NOT EXISTS` guards
  are idempotent but unversioned relative to the app. `prisma migrate deploy`
  gives a single source of truth and a deploy-time gate.
- **Portability.** Supabase is PostgreSQL; Prisma supports it natively, so this
  is a *layering* change (ORM on top of the same DB), not a *engine* change.
  The Supabase project, its URL, its auth, and its RLS all stay.

### What must be migrated (in order)
1. **`prisma/schema.prisma`** — model every table in
   `docs/infrastructure/4-supabase-provider.md` §Schema, including the
   `provider_services.raw JSONB` column and the `orders.payment_method` column
   (migration `20260916000001`).
2. **`prisma/migrations/`** — re-express the 30 Supabase SQL migrations as
   Prisma migrations, OR keep them as raw SQL run via
   `prisma.$executeQueryRaw` inside a single bootstrap migration. The atomic
   wallet functions (`credit_wallet` / `debit_wallet`, migration 11) must be
   created as SQL functions in the Prisma migration set.
3. **Replace the 4 Supabase client variants** (`src/lib/supabase/{client,server,
   middleware,admin}.ts`) with a single `src/lib/prisma.ts` exporting a
   per-request Prisma client (`lib/` for server components, `@prisma/client`
   adapter for edge where needed). **Supabase Auth is NOT replaced** — only the
   data-access layer. Auth still uses `@supabase/ssr` for sessions/cookies.
4. **Reimplement the wallet helpers** (`src/lib/wallet.ts`) against Prisma
   transactions instead of the `debit_wallet`/`credit_wallet` RPCs, OR keep the
   RPCs and call them via `prisma.$queryRaw`.
5. **Reimplement `listJanjezServices` pagination** (currently a manual
   `while(true) { query.range(from, to) }` loop, `janzez-services.ts:106`) with
   Prisma `skip`/`take`.
6. **Update tests** — `vitest.config.mts` must point at the Prisma test
   database; `src/lib/business-side/idempotency.ts` and `rate-limit.ts` are
   in-memory and unaffected.
7. **Update `package.json`** — add `@prisma/client` and `prisma` (dev), keep
   `@supabase/ssr` + `@supabase/supabase-js` for Auth.

### Explicitly NOT changed by this migration
- Supabase Auth (sessions, cookies, Google OAuth, email verification).
- Supabase Realtime (the `useNotifications` hook, `src/lib/supabase/realtime.ts`).
- Supabase Storage (avatars, migration 15).
- The M-Pesa Daraja integration, the DripFeedPanel SMM API, Brevo SMTP.
- The Business Side HMAC island (it calls `createAdminClient()` — swap the
  inner client, keep the auth layer).

### Rollback
Prisma migrations are additive and reversible via `prisma migrate resolve` /
`prisma migrate rollback`. Until the migration is complete, the Supabase raw
client remains the production path and Prisma is opt-in per-module.

### Ground truth for the current (pre-Prisma) state
`package.json` dependencies: `@supabase/ssr`, `@supabase/supabase-js`,
`@types/nodemailer`, `jose`, `next`, `nodemailer`, `react`, `react-dom`,
`sharp`. No Prisma. No other DB driver.