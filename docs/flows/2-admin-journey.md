# 2. Admin journey — service management to ledger

**Ground truth:** `src/app/admin/page.tsx`, `src/app/api/admin/services/route.ts`,
`src/app/api/admin/services/sync-prices/route.ts`, `src/app/api/admin/integrations/route.ts`,
`src/app/api/admin/ledger/route.ts`, `src/app/api/admin/orders/route.ts`,
`src/app/api/admin/orders/[id]/actions/route.ts`, `src/app/api/admin/blog/notifications/route.ts`,
`src/app/api/admin/users/route.ts`, `src/app/api/admin/contact/route.ts`,
`src/app/api/admin/contact/reply/route.ts`, `src/app/api/admin/fulfillment-logs/route.ts`,
`src/app/api/admin/provider-services/route.ts`, `src/app/api/admin/stats/route.ts`,
`src/app/api/admin/settings/drip-feed/route.ts`, `src/lib/server/auth-helpers.ts:48`.

## 2.1 Admin gate (two layers)
```
middleware: [ADMIN && user]
  ├─ user_metadata.role === "admin" → pass
  └─ else → SELECT role FROM profiles WHERE id=user.id
      └─ role === "admin" → pass; else pass through (client renders 403)

API route: requireAdmin(request)
  ├─ getUserFromRequest (Bearer / x-api-key → supabase.auth.getUser)
  ├─ !user → 401
  └─ role !== "admin" → 403
```
- **Promotion:** `scripts/promote-admin.mjs` + `GET /api/auth/check-admin/route.ts`
  returns a one-line SQL fix: `UPDATE public.profiles SET role='admin' WHERE id='<uuid>';`
  then sign-out and sign-in again.

## 2.2 Service management
| Route | Method | Purpose |
|---|---|---|
| `/api/admin/services` | GET / POST | list / create janjez_services |
| `/api/admin/services/[id]` | GET / PATCH / DELETE | read / update / soft-delete (is_active=false) |
| `/api/admin/services/sync-prices` | POST | bulk-update selling_price_ksh/min/max; **requires provider_service_id** |
| `/api/admin/services/bulk` | POST | bulk ops |
| `/api/admin/provider-services` | GET | read cached provider catalog |

- **Price derivation:** when creating a service with `provider_service_id` but no
  `selling_price_ksh`, the route auto-derives `rate * 13.7` from the provider
  service row (`src/app/api/admin/services/route.ts:92`). This `13.7` multiplier
  is the Janjez markup and is also in `pricing.ts:getJanjezSellingPrice`.
- **`provider_service_id` is stripped from customer-facing responses**
  (`toBusinessService` and the catalogue API projection) — it is an internal
  provider reference, never exposed to customers.

## 2.3 Integrations
`GET /api/admin/integrations` returns: provider name, whether `SMM_API_URL` /
`SMM_API_KEY` are configured, live `getProviderBalance()`, count of cached
provider services, and `last_catalog_sync`.

## 2.4 Ledger
`GET /api/admin/ledger?page=&limit=&type=` unions four sources into one timeline:
1. `orders` (type=order, category=status)
2. `wallet_transactions` (type=tx, category=status|payment_method)
3. `email_verifications` (type=verification)
4. `admin_activity_logs` (type=admin_action)

Sorted by `created_at` DESC, paginated. **Failures are never hidden** — the
default filter shows completed/pending/failed explicitly.

## 2.5 Orders & fulfillment actions
- `GET /api/admin/orders` — list (admin only).
- `GET/PATCH /api/admin/orders/[id]` — read / update status.
- `POST /api/admin/orders/[id]/actions` — trigger `fulfillOrder`, `refillOrder`,
  `cancelOrder`, `requestRefill`, `requestCancel`.
- `GET /api/admin/fulfillment-logs` — the `fulfillment_logs` audit table written
  by `logFulfillment()` on every provider interaction.
- `GET /api/admin/stats` — dashboard aggregates.
- `GET/PUT /api/admin/settings/drip-feed` — `platform_settings.drip_feed_limits`.
- `GET /api/admin/users` — user list.
- `GET /api/admin/blog/notifications` — admin notification feed.
- `GET /api/admin/contact` + `POST /api/admin/contact/reply` — contact form
  inbox with reply (reply is sent as a branded Brevo email via
  `getContactReplyHtml`).

## 2.6 Rationale
- **Admin writes always go through `requireAdmin`.** Middleware is a redirect
  gate, not an authorization gate — the API is the authority.
- **`provider_service_id` is stripped at two points:** `toBusinessService`
  (business-side API) and the customer catalogue API projection. This was a
  deliberate security fix (commit `5adfbec`, "remove provider_service_id and
  serviceId from customer-facing catalogue API").
