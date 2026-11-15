# A1 — Sign Up (email + password)

**Ground truth:** `src/components/AuthContext.tsx:360` (`customSignUp`),
`src/app/api/auth/send-verification/route.ts:63` (`POST`), `src/lib/email/mailer.ts`,
`src/lib/transactional.ts`.

## Entry
`src/components/auth/SignUpForm.tsx` → `useAuth().customSignUp(email, password, full_name?, phone?, username?)`.

## Flow

```
SignUpForm
  │
  ▼  POST /api/auth/send-verification  (fetchWithTimeout, no auth header)
  │   body: { email, password, full_name?, phone?, username?, resend? }
  │
  ├─ validate: email regex, password ≥ 6 chars, username ^[a-z0-9_]{3,30}$
  ├─ rateLimit(request, SIGNUP_RATE_LIMIT=10)        # src/lib/server/rate-limiter.ts
  │
  ├─ createAdminClient()   # SUPABASE_SERVICE_ROLE_KEY
  │
  ├─ supabase.auth.admin.listUsers()  → find existing user by email
  │   ├─ existing & email_confirmed_at  → 409 "already registered and verified"
  │   ├─ existing & !confirmed          → re-send verification, 200 "account exists but not verified"
  │   └─ not existing                   → continue
  │
  ├─ supabase.auth.admin.createUser({ email, password, email_confirm: false,
  │     user_metadata: { full_name, phone, username } })
  │   ├─ error → 400
  │   └─ success → user.id
  │
  ├─ (if username) supabase.from("profiles").update({ username }).eq(id)
  │   └─ error → deleteUser(id), 500 "Failed to set username"
  │
  └─ sendVerificationEmail(supabase, userId, email, fullName)
        ├─ crypto.randomBytes(32).toString("hex")  → token
        ├─ expires_at = now + 24h
        ├─ DELETE old tokens for this user_id
        ├─ INSERT email_verifications { user_id, token, expires_at }
        ├─ verifyUrl = `${SITE_URL}/api/auth/verify-email?token=${token}`
        └─ sendTransactional({ name: "user.verify_email", userId, email, fullName,
               data: { fullName, verifyUrl, expiresInHours: 24 } })
              ├─ email  → Brevo SMTP (src/lib/email/mailer.ts)
              └─ in-app notification (category: "security", severity: "info")
```

## Exit
`{ ok: true, message: "Account created. Check your email to verify." }`.
User is **NOT** signed in. They must click the verification link.

## Rationale (why the line is drawn here)
- **Custom verification, not Supabase built-in.** The README documents two
  options. Option A (Supabase dashboard SMTP) is the default. Option B (custom
  flow) is what is implemented: `email_confirm: false` on create, then a custom
  token + Brevo email. This gives the "JANJEZ SOCIO" branded sender identity.
  Commit `d9b427e` removed a now-dead `signUp()` from AuthContext precisely
  because it triggered broken Supabase email links — the custom flow is the
  authoritative one.
- **`email_confirm: false` means the user cannot sign in until verified.**
  The verify-email route flips `email_confirm: true` via `updateUserById`.
- **Rate limit is on the API, not the form.** The form itself has no client-side
  throttle; the server enforces it.

## Side effects
- `email_verifications` row (24h TTL).
- One Brevo email.
- One in-app notification.
- `profiles` row is **NOT** created here — it is created lazily by
  `ensureProfile()` in AuthContext on first session load (`src/components/AuthContext.tsx:50`).