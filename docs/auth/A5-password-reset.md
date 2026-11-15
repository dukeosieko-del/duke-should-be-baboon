# A5 — Password Reset

**Ground truth:** `src/app/api/auth/reset-password/route.ts:14` (`POST`),
`src/app/api/auth/set-password/route.ts:10` (`POST`),
`src/lib/auth/reset-helpers.ts`, `src/components/auth/ResetPasswordForm.tsx`.

## Two phases

### Phase 1 — request
```
POST /api/auth/reset-password  { email }
  ├─ rateLimit(5)
  ├─ EMAIL_RE validation
  ├─ createAdminClient()
  ├─ listUsers → find user by email
  │   └─ not found → 200 GENERIC_RESPONSE  (don't reveal existence)
  ├─ token = crypto.randomBytes(32).toString("hex")
  ├─ expires_at = now + 60 min
  ├─ INSERT password_reset_tokens { user_id, token, expires_at }
  ├─ resetUrl = `${SITE_URL}/auth/reset-password?token=${token}`
  └─ sendTransactional("user.password_reset", { resetUrl, expiresInMinutes: 60 })
```

### Phase 2 — set
```
POST /api/auth/set-password  { token, password }
  ├─ rateLimit(10)
  ├─ consumeResetToken(token)        # src/lib/auth/reset-helpers.ts
  │   └─ SELECT ... WHERE token=? AND expires_at > now AND used_at IS NULL
  │       → mark used_at, return { user_id } or null
  ├─ setPassword(user_id, password)  # supabase.auth.admin.updateUserById
  └─ sendTransactional("user.password_reset_confirmation")
```

## Exit
Phase 1: generic 200. Phase 2: `{ ok: true, message: "Password updated successfully" }`.

## Rationale
- **Generic response on unknown email** prevents user-enumeration.
- **Token is single-use + 60 min**, consumed atomically so a replayed token
  fails the `used_at IS NULL` check.
- **Password minimum is 6 chars**, matching signup.