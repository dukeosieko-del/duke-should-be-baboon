# 3. Brevo email transport

**Ground truth:** `src/lib/email/mailer.ts:43` (`getTransport`),
`src/lib/email/config.ts`, `src/lib/email/templates.ts`, `src/lib/transactional.ts:284`
(`sendTransactional`), commit `7d1fb3e` ("migrate from ZeptoMail to Brevo SMTP via nodemailer").

## Transport
- **Library:** nodemailer v9.
- **Host:** `BREVO_SMTP_HOST` (default `smtp-relay.brevo.com`), port `BREVO_SMTP_PORT`
  (default 587), `secure: false` (STARTTLS).
- **Auth:** `BREVO_SMTP_USER` (SMTP login, NOT the Brevo account email) +
  `BREVO_SMTP_KEY` (SMTP relay key, NOT the Brevo API key).
- **From:** `BREVO_FROM_EMAIL` (must be a verified Brevo sender) + `BREVO_FROM_NAME`
  (default "JANJEZ SOCIO").
- **Singleton transport**, cached in `cachedTransport`, with `verifyMailer()`.

## sendTransactional — the unified dispatch
Every email in the system goes through `sendTransactional(event)`:
1. Resolve template from `registry` (16 registered events).
2. Render subject/html/text/notificationTitle/notificationBody via the template.
3. Resolve recipient email: `event.email` → else `lookupEmailForUser(userId)`
   (SELECT email, full_name FROM profiles).
4. **Send email** via Brevo (`sendEmail`).
5. **Create in-app notification** via `notifyUser` (user audience) or
   `notifyAdmins` (admin audience), with a `createNotification` fallback.

## Registered events (16)
| Event | Audience | Category | Triggered by |
|---|---|---|---|
| `user.welcome` | user | system | verify-email success |
| `user.verify_email` | user | security | signup / resend |
| `user.password_reset` | user | security | reset request |
| `user.password_reset_confirmation` | user | security | set-password success |
| `user.security_alert` | user | security | sign-in |
| `order.received` | user | order | order created (wallet) |
| `order.completed` | user | order | fulfillment completed |
| `order.failed` | user | order | fulfillment failed |
| `payment.received` | user | wallet | M-Pesa callback |
| `wallet.low_balance` | user | wallet | debit below KES 100 |
| `admin.new_order` | admin | admin_alert | (reserved) |
| `admin.high_value_order` | admin | admin_alert | (reserved, >KES 5,000) |
| `admin.fulfillment_failure` | admin | admin_alert | fulfillment failure |
| `system.maintenance` | user | system | (reserved) |
| `admin.*` variants | admin | admin_alert | — |

## Templates
`src/lib/email/templates.ts` (1073 lines) renders every email with a shared
`shell()` wrapper: black header with the JANJEZ SOCIO wordmark + "Pata Clout
Chapchap", a 6px green/red/black accent bar, 600px max-width card body, and a
footer with social links. Brand colours: green `#00A859`, red `#BB133E`.

## Rationale
- **One dispatch path.** Every transactional email (verification, reset, welcome,
  order received/complete/failed, payment received, low balance, admin alerts)
  goes through `sendTransactional`. There is no direct `sendEmail` call outside
  this wrapper except the anonymous-order receipt and the M-Pesa callback
  payment-received email, which use the template helpers directly.
- **Brevo replaced ZeptoMail** because ZeptoMail's `ZEPTOMAIL_SENDMAIL_TOKEN`
  was invalid (TM_4001 Access Denied) and blocked all email-dependent auth
  flows. The migration is documented in `JANJEZ_BUILD_STATE.md` MILESTONE 5.