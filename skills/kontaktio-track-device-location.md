---
name: kontaktio-track-device-location
description: Find where a Kontakt.io-tracked device, asset or badge is right now, and reconstruct where it has been, using the Kio Cloud Location & Occupancy API.
api: Kontakt.io Location & Occupancy API
spec: openapi/kontaktio-location-occupancy-openapi.yml
base_url: https://apps.cloud.us.kontakt.io
operations:
  - get-locations-campuses
  - get-locations-buildings
  - get-locations-floors
  - get-locations-rooms
  - get-positions
  - get-positions-history
  - get-presences
  - get-presences-history
  - get-v3-novid-colocations
---

# Track a device's location

Every device is addressed by its **trackingId** (the device MAC address). Locations are
addressed by numeric ids arranged as campus > building > floor > room.

## Before you start

- Auth: send `Api-Key: <Kio Cloud Server API Key>` on every request. Get it from Kio Cloud >
  Users > Security > Server API Key. The Location & Occupancy spec declares no other scheme.
- Region matters. `https://apps.cloud.us.kontakt.io` for US accounts,
  `https://apps.cloud.uk.kontakt.io` for UK.
- `Content-Type: application/json` unless an endpoint says otherwise.

## Steps

1. **Resolve the location tree once and cache it.** Call `get-locations-campuses`
   (`GET /v2/locations/campuses`), then `get-locations-buildings`, `get-locations-floors`,
   `get-locations-rooms` as needed. Positions come back as ids, not names, so without this
   map the answer is unreadable.
2. **Get the current position.** `get-positions` (`GET /v2/positions`) returns the device's
   last known position: `x`, `y`, floor/room ids, plus `lost`, `moving` and `irAssisted`
   flags. Treat `lost: true` as "no fresh fix", not as "at the last coordinates".
3. **Get the history when the question is "where has it been".** `get-positions-history`
   (`GET /v2/positions/history`) accepts a time range of up to **7 days** (raised from 1 hour
   in the 2024-10 release).
4. **Use presence for room-level questions.** `get-presences` (`GET /v3/presences`) answers
   "which devices are in this location right now" — the inverse lookup of positions.
   `get-presences-history` (`GET /v3/presences/history`) gives the same over a time range.
5. **Use colocations for proximity questions.** `get-v3-novid-colocations`
   (`GET /v3/novid/colocations`) lists devices that were near each other.

## Rules that will bite you

- **Historical endpoints are rate limited to 30 requests/minute**, separately from the
  40 requests/second limit on everything else. `get-positions-history`,
  `get-presences-history` and `get-telemetry` are all in that bucket. On exhaustion you get
  **429 with a `Retry-After` header** — the header is sent twice, once as an HTTP-date and
  once as delay-seconds. Parse either, back off at least one second, and do not retry blind.
- There are **no rate-limit quota headers**. You cannot see how much budget is left; you only
  learn you are over by being refused.
- Errors are **not** RFC 9457. The body is `{message, errors[{field, error, invalidValue}]}`.
  See `errors/kontaktio-problem-types.yml`.
- **No 5xx response is declared anywhere in the spec.** Handle server failure defensively;
  the contract will not tell you what it looks like.

## Related artifacts

`conventions/kontaktio-conventions.yml` · `rate-limits/kontaktio-rate-limits.yml` ·
`data-model/kontaktio-data-model.yml`
