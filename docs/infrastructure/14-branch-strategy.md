# 14. Branch strategy & deployment gates

**Ground truth:** `JANJEZ_BUILD_STATE.md` §4 (BUILD LINEAGE), §16 (CHANGE CONTROL),
`.github/workflows/webpack.yml`, `DEPLOY.md`.

## Branch model

```
main                          ← production (Vercel auto-deploy on push)
  │
  ├─ feature/<short-desc>     ← new functionality (e.g. feature/prisma-migration)
  │
  ├─ fix/<short-desc>         ← bug fixes (e.g. fix/mpesa-callback-idempotency)
  │
  ├─ review/<tag>             ← reconciliation / audit branches
  │     (e.g. review/janjez-reconciliation-20260822)
  │
  └─ session/<uuid>           ← agent session branches (ephemeral)
```

**Rule (G5): never push to `main` unless explicitly authorized.**
PRs to `main` must pass CI (`webpack.yml`: test → typecheck → lint → build).

## Deployment gates (from CHANGE CONTROL)

| Stage | Authorization | What happens |
|---|---|---|
| DIAGNOSTIC | none | read-only inspection |
| AUTHORIZED CODE CHANGE | explicit | modify isolated worktree/file |
| VALIDATE | none (after change) | tests / build / static checks |
| COMMIT | explicit | `git commit` |
| PUSH | explicit | `git push` |
| DEPLOY | explicit | separate controlled operation |
| RUNTIME VALIDATION | explicit | verify staging |
| PROMOTE | explicit | separate operation |
| ROLLBACK | explicit | use recorded known-good build/commit |

## CI pipeline (`.github/workflows/webpack.yml`)
Runs on push/PR to `main` and every 5 minutes (cron):
```
npm ci  →  npm run test:run  →  npx tsc --noEmit  →  npm run lint  →  npm run build
```
Secrets injected: `NEXT_PUBLIC_SUPABASE_*`, `SUPABASE_SERVICE_ROLE_KEY`,
`SMM_API_*`, `CRON_SECRET`, `NEXT_PUBLIC_SITE_URL`, `BREVO_*`, `MPESA_*`.

## DB deployment target (directive, 2026-11-15)

**All future database deployments must target Prisma.**

- **Current path:** Supabase SQL migrations applied manually in the dashboard
  SQL Editor.
- **Future path:** `npx prisma migrate deploy` as a build/deploy step.
- **Migration spec:** `docs/infrastructure/12-prisma-migration-note.md`.
- **Not yet started.** The Supabase raw-SQL layer remains production until the
  Prisma migration is executed and verified.

## Rollback references
- Promoted BUILD_ID: `xnROBkKJlifhXYRMDlT91`
- Promoted commit: `896d081c37263cc4c73a13eddf35cacb72029943`
- Clean candidate branch: `clean-rebuild-20260819`
- Staging baseline: `b4274a8b60cd93be21802b638a13381916491cad`
- EC2 preservation: `/tmp/janjez-aug24-25.diff`, `/tmp/janjez-untracked.txt`,
  `/tmp/janjez-forensic-metadata.txt`