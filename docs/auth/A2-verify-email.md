# A2 — Email Verification (click link)

**Ground truth:** `src/app/api/auth/verify-email/route.ts:11` (`GET`).

## Entry
User clicks the Brevo email link → `GET /api/auth/verify-email?token=...`.

## Flow

```
GET /api/auth/verify-email?token=xxx
  │
  ├─ rateLimit(request, 60)
  ├─ token missing        → redirect /auth/sign-in?error=missing_token
  │
  ├─ createAdminClient()
  ├─ SELECT * FROM email_verifications WHERE token = xxx
  │   ├─ not found        → redirect /auth/sign-in?error=invalid_token
  │   └─ found            ▼
  ├─ expired? (now > expires_at)
  │   ├─ DELETE token
  │   └─ redirect /auth/sign-in?error=token_expired
  │
  ├─ SELECT id,email,email_verified FROM profiles WHERE id = verification.user_id
  │   └─ not found → redirect /auth/sign-in?error=profile_not_found
  ├─ already verified     → DELETE token, redirect /auth/sign-in?verified=1&email=...
  │
  ├─ supabase.auth.admin.updateUserById(user_id, { email_confirm: true })
  │   └─ error → redirect /auth/sign-in?error=verification_failed
  │
  ├─ sendTransactional({ name: "user.welcome", ... })   # welcome email
  ├─ DELETE email_verifications row
  └─ redirect /auth/sign-in?verified=1&email=...
```

## Exit
Redirect to sign-in with `verified=1`. User can now sign in.

## Rationale
- **Redirect, not JSON.** This is an email link; the server never returns a SPA
  payload. All terminal states are URL-query error codes so the sign-in page can
  show the right message.
- **Token is single-use + 24h TTL.** Deleted on success or expiry.