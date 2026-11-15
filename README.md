# duke-should-be-baboon

> **Surgical-repair documentation repository for the `janjez-socio` build.**
>
> Every framework flow, API provision, build step, and decision rationale of the
> `janjez.social` Next.js 16 + Supabase application is captured here as markdown
> so an agent can repair, extend, or rebuild any subsystem without re-discovery.

## Source of truth

| Item | Value |
|---|---|
| Source repo | `dukeosieko-del/janjez-socio` |
| Working copy analyzed | `/home/ubuntu/janjez-socio-mobile-worktree` |
| Branch | `main` (HEAD `b979f30`) |
| Commits | 225 |
| Framework | Next.js 16.2.10 (App Router, `output: "standalone"`), React 19.2.4 |
| Database provider | **Supabase (PostgreSQL)** — no Prisma, no other DB provider ever existed |
| ORM | none — raw `@supabase/supabase-js` + `@supabase/ssr` + hand-written PL/pgSQL migrations |
| Email transport | **Brevo SMTP** via nodemailer (migrated from ZeptoMail in commit `7d1fb3e`) |
| Payment | M-Pesa Daraja STK push (Safaricom) |
| SMM fulfillment provider | DripFeedPanel SMM API v2 |
| Deploy target | Vercel (auto-deploy on `main`) + EC2/PM2 standalone + nginx |

## How to read this repo

```
README.md                      <- this file
docs/
  flows/                      <- end-to-end user & system journeys (entry → fulfilment → exit)
  diagrams/                   <- ASCII / mermaid flow diagrams
  api/                        <- every route: method, path, auth, contract
  auth/                       <- Google OAuth vs email/password, sessions, middleware
  fulfillment/                <- SMM provider, drip-feed, retry, refund
  business-side/              <- HMAC island API (dripfeeds / orders / wallet / oauth)
  blog/                       <- blog + community system
  infrastructure/             <- build, deploy, migrations, guardrails
figma/                        <- placeholder tree references for Figma assets
```

Each document follows the same contract:

1. **Ground truth** — which file / commit is authoritative.
2. **Entry point** — where the flow begins.
3. **Step-by-step** — every hop, every branch, every fallback.
4. **Exit** — terminal state and side effects (email, notification, DB row).
5. **Rationale** — why the line was drawn where it was, and which path was *not* taken.

## DB provider note (explicit)

**Current state:** there is no Prisma in this codebase, and there never was.
**Future state (directive, 2026-11-15): all future DB deployments must target
Prisma.** See `docs/infrastructure/12-prisma-migration-note.md` for the migration
spec. Until that migration is executed, the Supabase raw-SQL layer is production.

- `grep -ri prisma` across the repo, `ez-business-side`, and `janjez-business-worktree`
  returns **zero** matches outside of unrelated build artifacts (`.tsbuildinfo`).
- The database layer has always been **Supabase PostgreSQL**, accessed through
  `@supabase/supabase-js` (client/server/admin variants) and 30 hand-written
  SQL migrations under `supabase/migrations/`.
- The only "provider change" that ever happened was:
  - **Email transport**: ZeptoMail → Brevo SMTP (commit `7d1fb3e`,
    `feat(email): migrate from ZeptoMail to Brevo SMTP via nodemailer`).
  - **Supabase project hostname**: `snkgkcdnmhqaejpqftxn.supabase.co` (now
    NXDOMAIN/dead) → `rousjavuooduvicaobuv.supabase.co` (current, documented in
    `JANJEZ_BUILD_STATE.md` §6).
  - **No database engine or ORM swap.** If a future migration to Prisma happens,
    that is a *new* event, not a reconstruction of a past one.

## Build pipeline

```text
npm ci
  npm run build   # next build  (output: "standalone")
  npm run postbuild  # cp public/ + .next/static into .next/standalone/
  npm start       # next start --hostname 0.0.0.0  (PM2 via ecosystem.config.js)
```

See `docs/infrastructure/build.md` for the full pipeline, including the
`postbuild` script that was added in MILESTONE 4 to fix the missing-static-assets
regression.

## Guardrails (from `JANJEZ_BUILD_STATE.md` §15)

G1–G18 govern every operation: never confuse Cloud Agent workspace with EC2
runtime, every edit → test → record → commit → update docs, never commit
multiple unrelated changes, never deploy because a commit exists, etc.