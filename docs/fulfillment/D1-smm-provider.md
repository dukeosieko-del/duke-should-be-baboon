# D1 — SMM Provider layer (DripFeedPanel)

**Ground truth:** `src/lib/smm/provider.ts:1` (`SMM_API_URL`, `SMM_API_KEY`),
`src/lib/smm/fulfillment.ts`.

## Provider contract
- **Provider:** DripFeedPanel (SMM API v2).
- **Endpoint:** `SMM_API_URL` env, default `https://dripfeedpanel.com/api/v2`.
- **Auth:** `key` form parameter (`SMM_API_KEY` env).
- **Transport:** `POST application/x-www-form-urlencoded`, 30s timeout, AbortController.

## API surface (`smmPost` → `src/lib/smm/provider.ts:38`)

| Action | Params | Returns |
|---|---|---|
| `services` | — | `ProviderService[]` |
| `add` | service, link, quantity, runs?, interval? | `{ order?: number, error? }` |
| `status` | order | `{ status, charge, start_count, remains, currency }` |
| `status` | orders (comma-joined) | `Record<orderId, ProviderStatusResponse>` |
| `refill` | orders | `ProviderOrderResponse[]` |
| `cancel` | orders | `ProviderOrderResponse[]` |
| `balance` | — | `{ balance, currency }` |

`ProviderService` shape: `{ service: number, name, type, category, rate, min, max, refill, cancel, dripfeed }`.

## Caching
`provider_services` table is the cache. `syncProviderCatalog()` (admin POST
`/api/smm/catalog`) upserts the provider's service list, sets `is_active=false`
for any provider service the provider no longer returns, and stamps
`fetched_at`. `getProviderServices()` reads the cache (never the live API for
customer-facing reads).

## Rationale
- **Never call the live provider API from a customer request path.** All
  customer reads go through the `provider_services` cache; only admin sync and
  the cron job call the live API.
- **`dripfeed` flag is the provider's own capability flag**, distinct from
  Janjez's `supports_drip_feed` on `janjez_services`. Both must be true for a
  drip-feed order to be placed.