# 12. Prisma / DB-provider migration note

## Finding: Prisma was never used
`grep -ri prisma` across `/home/ubuntu/janjez-socio-mobile-worktree`,
`/home/ubuntu/ez-business-side`, and `/home/ubuntu/janjez-business-worktree`
returns **zero** source matches (only unrelated `.tsbuildinfo` build artifacts).

The database layer has **always** been **Supabase (PostgreSQL)** via
`@supabase/supabase-js` + `@supabase/ssr` + 30 hand-written SQL migrations.

## The only "provider changes" that ever happened
1. **Email transport: ZeptoMail → Brevo SMTP** (commit `7d1fb3e`,
   `feat(email): migrate from ZeptoMail to Brevo SMTP via nodemailer`).
   ZeptoMail's token was invalid (TM_4001 Access Denied) and blocked all
   email-dependent auth flows.
2. **Supabase project hostname** changed from
   `snkgkcdnmhqaejpqftxn.supabase.co` (now NXDOMAIN) to
   `rousjavuooduvicaobuv.supabase.co` — a manual `.env` reconstruction, not a
   code change (JANJEZ_BUILD_STATE.md §6).
3. **No database engine or ORM swap.** If a future migration to Prisma happens,
   that is a *new* event, not a reconstruction of a past one.

## If a Prisma migration is introduced going forward
- The migration would add `@prisma/client` to `package.json`, a `prisma/schema.prisma`
  file, and `prisma/migrations/` — none of which exist today.
- The existing 30 Supabase SQL migrations would need to be re-expressed as Prisma
  migrations or kept as raw SQL executed via `prisma.$executeQueryRaw`.
- The four Supabase client variants (`client.ts`, `server.ts`, `middleware.ts`,
  `admin.ts`) would need to be replaced or wrapped.
- The atomic wallet RPCs (`credit_wallet` / `debit_wallet`) would need to be
  reimplemented as Prisma transactions or kept as raw SQL.

## Ground truth for the current state
`package.json` dependencies: `@supabase/ssr`, `@supabase/supabase-js`,
`@types/nodemailer`, `jose`, `next`, `nodemailer`, `react`, `react-dom`,
`sharp`. No Prisma. No other DB driver.
