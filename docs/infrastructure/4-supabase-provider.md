# 4. Supabase — the database provider

**Ground truth:** `src/lib/supabase/{client,server,admin,middleware}.ts`,
`supabase/migrations/*.sql` (30 migrations), `src/lib/wallet.ts`.

## Provider
**Supabase PostgreSQL.** There is no Prisma, no other ORM, no other database.
`@supabase/supabase-js` + `@supabase/ssr` are the only DB clients.

## The four clients (never mix)
| Variant | File | Key | Use |
|---|---|---|---|
| Browser singleton | `client.ts` | `NEXT_PUBLIC_SUPABASE_URL/ANON_KEY` | client components, `useAuth`, realtime |
| Request SSR | `server.ts` | anon | server components (`cookies()` from `next/headers`) |
| Middleware SSR | `middleware.ts` | anon | edge middleware (try/catch wrapped) |
| Service-role admin | `admin.ts` | `SUPABASE_SERVICE_ROLE_KEY` | **all writes** |

**Invariant:** only the service-role client bypasses RLS and is permitted to
create users, confirm emails, and move money. It is only importable from
`src/app/api/**`, `src/lib/smm/**`, `src/lib/business-side/**` (all server-only).

## Atomic wallet RPCs
`supabase/migrations/20250101000011_wallet_atomic_functions.sql` defines:
- `credit_wallet(p_user_id UUID, p_amount DECIMAL) RETURNS TABLE(new_balance DECIMAL)`
- `debit_wallet(p_user_id UUID, p_amount DECIMAL) RETURNS TABLE(new_balance DECIMAL, success BOOLEAN)`

Both are `RETURNS TABLE(...)`, so PostgREST returns an **array** of rows. The
wrapper in `src/lib/wallet.ts:27` (`readWalletRpcRow`) explicitly unwraps
`data[0]`. A historical bug read `data.new_balance` directly on the array, which
returned `undefined` — making a *failed* debit look successful and every top-up
report a balance of 0. This unwrap is the fix.

## Schema (key tables)
| Table | Migration | Notes |
|---|---|---|
| `profiles` | `00_create_profiles_table` | `id UUID PK → auth.users`, `wallet_balance`, `role`, `email_verified`, `username`, `last_low_balance_email_at` |
| `orders` | `03_orders_table` + `05_orders_extended` + `08/09_orders_order_id` + `34_orders_retry_and_status` + `20260916000001_orders_payment_method` | `payment_status` CHECK: `unpaid|pending_mpesa|paid|refunded` (added by `23_pending_mpesa_payment_status`) |
| `wallet_transactions` | `04_wallet_transactions` | `reference` = CheckoutRequestID, `related_order_id` |
| `janjez_services` | `22_service_placement` | placement flags: `show_sidebar|show_landing|show_guarded|show_anonymous|show_catalogue` |
| `provider_services` | `07_smm_provider_integration` | cached DripFeedPanel catalog |
| `email_verifications` | `01_email_verification` | custom verification tokens |
| `password_reset_tokens` | `24_password_reset_tokens` | single-use, 60 min |
| `notifications` | `06_notifications` + `28_notifications` + `30_notifications_relax_legacy` | `audience` (user|admin), `category`, `severity`, `read_at` |
| `fulfillment_logs` | `10_fulfillment_logs_provider_order_id_index` | audit trail |
| `blog_categories` / `blog_posts` / `blog_tags` / `blog_post_tags` / `blog_comments` / `blog_ratings` / `blog_reports` | `31_blog_community_tables` | |
| `contact_messages` + reply | `32_contact_messages` + `33_contact_messages_reply` | |
| `security_events` | `27_security_events` | |
| `platform_settings` | (key/value) | `drip_feed_limits` |
| `admin_activity_logs` | `02_admin_tables` | |
| `affiliate` | `14_affiliate` | |
| `storage_avatars` | `15_storage_avatars` | |

## Rationale
- **No Prisma.** The build never used Prisma. If a future migration introduces
  it, that is a new event. All 30 migrations are additive SQL with `IF NOT EXISTS`
  / `DROP CONSTRAINT IF EXISTS` guards so they are idempotent (commit
  `11259f0`, "make migration idempotent by adding drop statements").
- **RLS is on every table.** `profiles`, `orders`, `provider_services`,
  `contact_messages` all have row-level policies. The service-role client
  bypasses them, which is why the service-role key must never reach the browser.