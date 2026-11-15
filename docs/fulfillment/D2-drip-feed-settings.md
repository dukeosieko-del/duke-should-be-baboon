# D2 — Drip-Feed settings

**Ground truth:** `src/lib/drip-feed-settings.ts:1` (`getDripFeedLimits`),
`platform_settings` table, `src/app/api/admin/settings/drip-feed/route.ts`.

## Configuration source
`SELECT value FROM platform_settings WHERE key = 'drip_feed_limits'`
Returns a JSON value:
```json
{ "enabled": true, "min_runs": 1, "max_runs": 20,
  "min_interval": 10, "max_interval": 1440 }
```
- `enabled` — global drip-feed toggle.
- `min_runs` / `max_runs` — how many repeat runs the customer may request.
- `min_interval` / `max_interval` — minutes between runs (10 min … 24 h).

**Fallback if DB missing/unconfigured:**
`{ enabled: true, min_runs: 1, max_runs: 20, min_interval: 10, max_interval: 1440 }`.

`getDripFeedLimitsSync()` returns the fallback synchronously (used where an
async DB call is not possible).

## Where drip-feed is enforced

1. **Order validation** (`src/app/api/orders/route.ts:217`, anonymous route:96):
   if `runs` or `interval` is supplied, the service must have
   `supports_drip_feed = true`, else 400 "This service does not support drip-feed".
2. **Fulfillment** (`src/lib/smm/fulfillment.ts:309`): `placeProviderOrder`
   forwards `runs` and `interval` to the provider's `add` action when present.
3. **Provider capability:** the provider service row must also have
   `dripfeed === true` (mapped to `supports_drip_feed` on the cached
   `provider_services` row).

## Admin surface
`GET/PUT /api/admin/settings/drip-feed` reads/writes `platform_settings`.

## Rationale
- **Drip-feed is a customer-facing feature, not a provider detail.** The
  `janjez_services.supports_drip_feed` flag is what the order API checks; the
  provider's own `dripfeed` flag is what the fulfillment layer checks. A service
  can be marketed as drip-feed-capable only if both agree.
- **Limits are global, not per-service.** This keeps the admin surface simple and
  prevents a per-service limit table from drifting out of sync with the
  provider catalog.