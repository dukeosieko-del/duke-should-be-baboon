# A6 — Middleware, Sessions, and Authorization

**Ground truth:** `src/middleware.ts:7`, `src/lib/server/auth-helpers.ts`,
`src/lib/auth/protected-route.tsx`, `src/lib/supabase/{middleware,server,client,admin}.ts`.

## Three Supabase clients — never mix them

| Variant | File | Key used | Purpose |
|---|---|---|---|
| Browser singleton | `src/lib/supabase/client.ts` | `NEXT_PUBLIC_SUPABASE_*` | Client components, `useAuth()`, realtime |
| Request-scoped SSR | `src/lib/supabase/server.ts` | `NEXT_PUBLIC_SUPABASE_*` | Server components, `cookies()` from `next/headers` |
| Middleware SSR | `src/lib/supabase/middleware.ts` | `NEXT_PUBLIC_SUPABASE_*` | Edge middleware, wrapped in try/catch |
| Service-role admin | `src/lib/supabase/admin.ts` | `SUPABASE_SERVICE_ROLE_KEY` | **All write paths** (API routes, fulfillment, cron) |

**Invariant:** the service-role client is the *only* one that bypasses RLS and
is the only one allowed to create users, confirm emails, and credit wallets.
It is never imported by a client component (it is only reachable from `src/app/api/**`
and `src/lib/smm/**`, `src/lib/business-side/**`, which are server-only).

## Middleware matcher (`src/middleware.ts:102`)
```
matcher = [
  "/dashboard/:path*",
  "/auth/:path*",
  "/admin", "/admin/:path*",
  "/orders/:path*",
  "/pay",
  "/wallet/:path*",
  "/settings/:path*",
]
```

## Decision tree per request
```
middleware(request)
  ├─ createClient() (middleware variant)
  │   └─ null (env missing) → in prod: 500 JSON; in dev: pass through
  │
  ├─ getUser()  → user | null
  ├─ pathname
  │
  ├─ isAuthPage      = startsWith("/auth/")
  ├─ isAdminPage     = startsWith("/admin")
  ├─ isPublicTracking= startsWith("/orders/track")
  ├─ isProtectedPage = admin OR /dashboard OR (/orders && !track) OR
  │                     /pay OR /wallet OR /settings OR /blog/write OR
  │                     /blog/drafts OR /blog/my-articles OR (/blog/ && /edit)
  │
  ├─ [PROTECTED && !user] → redirect /auth/sign-in?next=<path>
  │
  ├─ [ADMIN && user]
  │   ├─ user_metadata.role === "admin"  → pass
  │   └─ else → SELECT role FROM profiles WHERE id=user.id
  │       ├─ role === "admin" → pass
  │       └─ else → pass through (403 is rendered client-side)
  │
  ├─ [AUTH PAGE && user] → redirect <returnTo> (default /services)
  │
  └─ otherwise → NextResponse.next()
```
Every response gets security headers (`applySecurityHeaders`) and, except for
`/_next/static/`, `Cache-Control: no-store, no-cache, must-revalidate, max-age=0, private`.

## Client-side admin gate
`src/lib/auth/protected-route.tsx` is a *fallback* — it pushes to
`/auth/sign-in?next=` when `!loading && !user`. The real admin gate is
`requireAdmin(request)` in `src/lib/server/auth-helpers.ts:48`:
```
getUserFromRequest(request)   # Bearer / x-api-key → supabase.auth.getUser(token)
  → SELECT id,email,role FROM profiles WHERE id=...
requireAdmin:
  !user           → 401 Unauthorized
  role !== "admin" → 403 Forbidden — admin access required
```
`requireCronSecret` checks `x-cron-secret` / `Authorization: Bearer <CRON_SECRET>`
against `process.env.CRON_SECRET`.

## Sessions
- Supabase Auth manages the JWT (access + refresh) in `sb-*` cookies.
- `@supabase/ssr` `cookies()` `setAll` writes them back on every server render.
- `AuthContext` keeps an in-memory `session` + `profile` + `walletBalance` +
  `isAdmin` (derived from `profiles.role`).
- On `SIGNED_OUT`, all `sb-*` localStorage keys and sessionStorage are wiped
  (`AuthContext.tsx:198`).

## Rationale
- **Middleware never grants admin.** It only *lets admin pages render*; the
  actual 403 comes from the API route. This avoids a security hole where a
  forged `user_metadata.role` could bypass the API.
- **`/orders/track` is explicitly public** — tracking links are shared.
- **`/auth/*` pages are excluded from the protected list** so the redirect loop
  (commit `413e625`, "remove isAuthPage from protected routes") cannot fire on
  the sign-in page itself.