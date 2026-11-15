# A3 — Sign In (email/password)

**Ground truth:** `src/components/AuthContext.tsx:302` (`signIn`),
`src/middleware.ts:73` (already-authenticated redirect).

## Entry
`src/components/auth/SignInForm.tsx` → `useAuth().signIn(email, password)`.

## Flow
```
supabase.auth.signInWithPassword({ email, password })
  ├─ success + session.access_token
  │   └─ POST /api/auth/security-alert  (fire-and-forget, Bearer token)
  │        → sendTransactional("user.security_alert")  # email + in-app
  └─ error → return { error }
```
- AuthContext `onAuthStateChange` listener updates `user`, `session`, `profile`
  (`loadProfile` → `ensureProfile` creates the `profiles` row lazily if missing).
- 30-minute inactivity timer auto-signs-out (`INACTIVITY_TIMEOUT_MS`).

## Exit
Session set; user lands on `returnTo` (from `?next=`) or `/services`.

## Rationale
- **Security alert is best-effort.** A failure here must never break sign-in,
  hence the try/catch swallowing.
- **`detectSessionInUrl: false`** on the client (`src/lib/supabase/client.ts:18`)
  because the app uses a *custom* verification flow, not Supabase's URL-hash flow.