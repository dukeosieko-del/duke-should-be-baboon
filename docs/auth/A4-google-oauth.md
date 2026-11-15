# A4 — Google OAuth

**Ground truth:** `src/components/AuthContext.tsx:325` (`signInWithOAuth`),
`src/components/auth/GoogleAuthButton.tsx`, `AUTH_SETUP.md` §"Google OAuth Setup".

## Entry
`GoogleAuthButton` → `useAuth().signInWithOAuth("google")`.

## Flow
```
supabase.auth.signInWithOAuth({
  provider: "google",
  options: {
    redirectTo: `${canonicalOrigin}/auth/callback?next=...`,
    queryParams: { access_type: "offline", prompt: "consent" },
  },
})
  ├─ error → { error }
  └─ data.url → window.location.assign(data.url)   # full-page redirect to Supabase → Google
```
- `canonicalOrigin` = `NEXT_PUBLIC_SITE_URL` (stripped of trailing slash) or
  `window.location.origin`. `next` is read from the current query string.
- After Google consent, Supabase redirects to `/auth/callback` which Supabase's
  SSR client resolves (cookies set by `@supabase/ssr` middleware).
- `ensureProfile()` runs on the resulting `SIGNED_IN` event, creating the
  `profiles` row with `email_verified: !!user.email_confirmed_at`.

## Exit
Authenticated session; profile ensured.

## Rationale
- **Google is the only OAuth provider.** The type is hardcoded `provider as "google"`.
- **`prompt: "consent"`** forces a refresh token so `access_type: "offline"` is
  meaningful; this is what makes long-lived sessions possible.
- **The Google client itself is configured in the Supabase dashboard**
  (`AUTH_SETUP.md`), not in code. Code only supplies the redirect URL.