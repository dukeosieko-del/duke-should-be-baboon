# 5. Build pipeline

**Ground truth:** `package.json` scripts, `next.config.ts`, `postcss.config.mjs`,
`ecosystem.config.js`, `.github/workflows/webpack.yml`, commit MILESTONE 4
(`postbuild` script).

## Commands
```json
"dev":     "next dev"
"build":   "next build"
"postbuild": "cp -r public .next/standalone/public/ && cp -r .next/static .next/standalone/.next/static && ..."
"start":   "next start --hostname 0.0.0.0"
"lint":    "eslint"
"test":    "vitest"
```

## The `postbuild` fix (MILESTONE 4)
Next.js `output: "standalone"` produces `.next/standalone/server.js` but **does
not** copy `public/` or `.next/static/` into the standalone tree. The result was:
- All 52 client JS chunks missing → no interactivity, no hydration.
- All images 404 → site appeared unstyled.

The `postbuild` script copies `public/` → `.next/standalone/public/` and
`.next/static/` → `.next/standalone/.next/static/` (plus the diagnostic/manifest
files). Every `npm run build` now self-packages the standalone output.

## next.config.ts
- `output: "standalone"`, `reactStrictMode: true`, `compress: true`.
- `productionBrowserSourceMaps: false`.
- `experimental.optimizePackageImports: ["lucide-react", "date-fns"]`.
- **Images:** `remotePatterns: [{ protocol: "https", hostname: "**" }]` (all
  remote hosts allowed), formats `["image/avif", "image/webp"]`,
  `minimumCacheTTL: 86400`, deviceSizes up to 3840.
- **Headers:** `/_next/static/:path*` → immutable 1-year cache; `/icons/:path*`
  and `/images/:path*` → 1-day + stale-while-revalidate; `/(.*)` →
  `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options: SAMEORIGIN`,
  `Permissions-Policy`.

## CI (`.github/workflows/webpack.yml`)
Triggers on `push`/`pull_request` to `main` and a `schedule: "*/5 * * * *"`
cron. Steps: checkout → setup-node 22 → `npm ci` → `npm run test:run` →
`npx tsc --noEmit` → `npm run lint` → `npm run build`. All secrets injected
via `${{ secrets.* }}`.

## PM2 (`ecosystem.config.js`)
`pm2 start ecosystem.config.js` runs `next start --hostname 0.0.0.0` on port
3000, with `instances: 1`, `autorestart: true`, and env loaded from `.env`
via `f9d7eb4` ("load .env vars in ecosystem.config.js without hardcoding secrets").

## Nginx
`nginx-janjez.conf` reverse-proxies to PM2 on `127.0.0.1:3000`. `nginx.conf`
is the global server config.

## Docker
`Dockerfile` + `docker-compose.yml` + `docker-compose.override.yml` provide the
containerised path (DEPLOY.md Option A).

## Rationale
- **Standalone is the only supported output.** Vercel uses its own builder, but
  the EC2/PM2 path requires standalone. The `postbuild` script is the guard
  that makes standalone actually runnable.
- **`hostname: 0.0.0.0`** is required for PM2 inside a container/EC2 to bind to
  all interfaces.