# 11. Deployment & infrastructure

**Ground truth:** `DEPLOY.md`, `ecosystem.config.js`, `docker-compose.yml`,
`Dockerfile`, `nginx-janjez.conf`, `ec2-deploy.sh`, `scripts/deploy-to-lightsail.sh`,
`janjez-socio.service`, `JANJEZ_BUILD_STATE.md` §13, §17.

## Targets
| Target | Mechanism | Repo |
|---|---|---|
| Vercel (production) | auto-deploy on `main` push | `dukeosieko-del/janjez-socio` |
| EC2 staging | PM2 + nginx, port 3000 | `/home/ubuntu/janjez-socio` |
| Docker | `docker compose up -d --build` | repo root |

## Environment variables (full set)
```env
# Site
NEXT_PUBLIC_SITE_URL=https://janjez.social

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://rousjavuooduvicaobuv.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=sb_publishable_...
SUPABASE_SERVICE_ROLE_KEY=            # NEVER in browser

# Brevo SMTP
BREVO_SMTP_HOST=smtp-relay.brevo.com
BREVO_SMTP_PORT=587
BREVO_SMTP_USER=                     # SMTP login, not Brevo email
BREVO_SMTP_KEY=                      # SMTP relay key, not API key
BREVO_FROM_EMAIL=                    # verified Brevo sender
BREVO_FROM_NAME=

# M-Pesa
MPESA_CONSUMER_KEY=
MPESA_CONSUMER_SECRET=
MPESA_PASSKEY=
MPESA_SHORTCODE=
MPESA_ENV=production                 # or "sandbox"
MPESA_CALLBACK_URL=                  # optional override

# SMM provider
SMM_API_URL=https://dripfeedpanel.com/api/v2
SMM_API_KEY=
SMM_FULFILLMENT_ENABLED=true

# Misc
NEXT_PUBLIC_GA_MEASUREMENT_ID=G-XXXXXXXXXX   # currently EMPTY — GA not implemented
CRON_SECRET=                         # for /api/cron/smm-sync
DEPLOY_URL=https://janjez.social
BUSINESS_SIDE_API_KEY=
BUSINESS_SIDE_HMAC_SECRET=
JANJEZ_MAIN_API_SECRET=
BUSINESS_SIDE_ACCOUNT_ID=
```

## M-Pesa sandbox vs production
`MPESA_ENV` selects the base URL:
- `production` → `https://api.safaricom.co.ke`
- anything else → `https://sandbox.safaricom.co.ke`

`PartyA` (`1329148`) and `PartyB` (`1799086`) in `initiateStkPush` are
**hardcoded sandbox values** and must be replaced for production. This is a
known gap flagged in the build state.

## Supabase hostname change
The original project `snkgkcdnmhqaejpqftxn.supabase.co` is now NXDOMAIN.
The current project is `rousjavuooduvicaobuv.supabase.co`. This was a manual
`.env` reconstruction (JANJEZ_BUILD_STATE.md §6), not a code change — the URL
lives in env only.

## Migrations
30 SQL files under `supabase/migrations/`, applied via the Supabase dashboard SQL
Editor. They are idempotent (`IF NOT EXISTS`, `DROP CONSTRAINT IF EXISTS`).
The authoritative schema is `docs/infrastructure/4-supabase-provider.md` §Schema.

## DNS
- Nameservers: `dayana.ns.cloudflare.com`, `noah.ns.cloudflare.com`
- A: `janjez.social` → `13.48.195.81`, `staging.janjez.social` → `13.48.195.81`
- CNAME: `www.janjez.social` → `janjez.social`

## CI/CD
`.github/workflows/webpack.yml` runs on push/PR to `main` and every 5 minutes
(cron). Steps: `npm ci` → `npm run test:run` → `npx tsc --noEmit` →
`npm run lint` → `npm run build`.

## Future DB deployment target: Prisma

**Directive (2026-11-15): all future database deployments must target Prisma.**

This is a forward-looking directive. The current production path is the Supabase
raw-SQL layer (30 migrations + 4 client variants). Prisma is the approved
future state and is **not yet implemented**.

Deployment flow when Prisma is adopted:
```bash
npx prisma migrate deploy      # applies prisma/migrations/ to the DB
npm run build                  # Next.js build
npm run postbuild              # standalone packaging
pm2 restart janjez-app         # or `next start` via PM2
```
Until then, DB changes are applied by running the Supabase SQL migrations
manually in the Supabase dashboard SQL Editor.

See `docs/infrastructure/12-prisma-migration-note.md` for the full migration
spec, including what is explicitly NOT changed (Supabase Auth, Realtime,
Storage, M-Pesa, DripFeedPanel, Brevo, Business Side HMAC).

## Rationale
- **Standalone output + `postbuild`** is the only path that works on EC2/PM2.
- **`next start --hostname 0.0.0.0`** is required for PM2 in a container.
- **GA is deliberately not implemented** — `NEXT_PUBLIC_GA_MEASUREMENT_ID` is
  empty and no analytics architecture exists (JANJEZ_BUILD_STATE.md MILESTONE 1).