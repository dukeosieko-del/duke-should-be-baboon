# 8. API route index

Every route in the app. **Auth column:** `none` = public, `anon` = Supabase anon
cookie (middleware-protected page), `Bearer` = `getUserFromRequest` (anon token
verified via service-role `getUser`), `admin` = `requireAdmin`,
`hmac` = `authenticateBusinessSideRequest`, `cron` = `requireCronSecret`.

## Auth
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/auth/send-verification` | none | signup / resend (rate-limited) |
| GET | `/api/auth/verify-email` | none | click email link → confirm |
| POST | `/api/auth/reset-password` | none | request reset token (rate-limited) |
| POST | `/api/auth/set-password` | none | consume token + update password |
| POST | `/api/auth/security-alert` | Bearer | fire-and-forget on sign-in |
| GET | `/api/auth/check-admin` | anon | diagnose + one-line admin SQL fix |

## Orders
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/orders` | Bearer | create order (wallet / mpesa) |
| GET | `/api/orders` | Bearer | list my orders |
| POST | `/api/orders/anonymous` | none | guest checkout |
| POST | `/api/cron/smm-sync` | cron\|admin | status sync |

## M-Pesa
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/mpesa/stk-push` | Bearer | initiate STK push |
| POST | `/api/mpesa/callback` | none | Safaricom callback |
| GET\|POST | `/api/mpesa/check-status` | Bearer | poll status + credit |

## Services (customer)
| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/services/catalogue` | none | DB-driven catalogue, `?placement=` filter |
| GET | `/api/services/sidebar` | none | sidebar placement |

## SMM (admin)
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/smm/catalog` | admin | sync provider catalog |
| GET | `/api/smm/sync-status` | admin | sync status |
| GET | `/api/smm/catalog` (alt) | admin | catalog read |

## Notifications
| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/notifications` | Bearer | list (cursor pagination, `?audience=`) |
| PATCH | `/api/notifications?action=mark-all-read` | Bearer | mark all read |
| PATCH | `/api/notifications/[id]` | Bearer | mark one read |
| DELETE | `/api/notifications/[id]` | Bearer | delete |

## Blog (public)
| Method | Path | Auth |
|---|---|---|
| GET | `/api/blog/posts` | none |
| GET | `/api/blog/posts/[slug]` | none |
| POST | `/api/blog/posts/[slug]/comments` | Bearer |
| POST | `/api/blog/posts/[slug]/ratings` | Bearer |
| POST | `/api/blog/comments/[id]/report` | Bearer |
| GET | `/api/blog/categories` | none |
| GET | `/api/blog/tags` | none |

## Admin
| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET\|POST | `/api/admin/services` | admin | service CRUD |
| GET\|PATCH\|DELETE | `/api/admin/services/[id]` | admin | service read/update/soft-delete |
| POST | `/api/admin/services/bulk` | admin | bulk ops |
| POST | `/api/admin/services/sync-prices` | admin | bulk price sync |
| GET | `/api/admin/provider-services` | admin | cached provider catalog |
| GET\|POST | `/api/admin/orders` | admin | list / create |
| GET\|PATCH | `/api/admin/orders/[id]` | admin | read / update |
| POST | `/api/admin/orders/[id]/actions` | admin | trigger fulfill/refill/cancel |
| GET | `/api/admin/fulfillment-logs` | admin | audit trail |
| GET | `/api/admin/ledger` | admin | unified financial timeline |
| GET | `/api/admin/stats` | admin | dashboard aggregates |
| GET\|PUT | `/api/admin/settings/drip-feed` | admin | drip-feed limits |
| GET | `/api/admin/users` | admin | user list |
| GET | `/api/admin/integrations` | admin | provider status |
| GET | `/api/admin/blog/posts` \| `/[id]` | admin | blog CRUD |
| GET\|POST | `/api/admin/blog/categories` \| `/tags` \| `/authors` | admin | taxonomy |
| GET | `/api/admin/blog/comments` \| `/ratings` \| `/reports` \| `/notifications` | admin | moderation |
| GET\|POST | `/api/admin/contact` \| `/reply` | admin | contact inbox + reply |

## Business Side
| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/business/v1/health` | hmac | liveness |
| GET | `/api/business/v1/services` \| `/[id]` | hmac | service list |
| POST | `/api/business/v1/orders` | hmac + idempotency-key | create order |
| GET | `/api/business/v1/orders/[id]` | hmac | read |
| POST | `/api/business/v1/orders/[id]/cancel` \| `/refill` | hmac | cancel/refill |
| GET | `/api/business/v1/wallet/balance` | hmac | balance |
| POST | `/api/business/v1/wallet/topup` | hmac + idempotency-key | STK top-up |
| POST | `/api/business/v1/oauth/token` | hmac | OAuth code → user |
| POST | `/api/business/v1/webhooks/register` | hmac | webhook registration |