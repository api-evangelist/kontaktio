---
name: kontaktio-monitor-room-occupancy
description: Read current and historical room, seat and footfall occupancy from Kio Cloud, and map each reading back to a named room, floor and building.
api: Kontakt.io Location & Occupancy API
spec: openapi/kontaktio-location-occupancy-openapi.yml
base_url: https://apps.cloud.us.kontakt.io
operations:
  - get-locations-rooms
  - get-v2-locations-room-sensors
  - get-v2-locations-spaces
  - get-v3-occupancy-rooms
  - get-v2-occupancy-roomId
  - get-v3-occupancy-room-attributes
  - get-v3-occupancy-room-attributes-history
  - get-v3-occupancy-spaces
  - get-v3-occupancy-spaces-history
  - get-v3-locations-images
---

# Monitor occupancy

Kio Cloud counts occupancy at three different grains and they are separate endpoints, not
parameters:

| Grain | Current | History |
|---|---|---|
| Room | `get-v3-occupancy-rooms` (`GET /v3/occupancy/rooms`) | `get-v2-occupancy-roomId` (`GET /v3/occupancy/rooms/history`) |
| Seat (room attribute) | `get-v3-occupancy-room-attributes` | `get-v3-occupancy-room-attributes-history` |
| Footfall space | `get-v3-occupancy-spaces` | `get-v3-occupancy-spaces-history` |

## Steps

1. **Build the location map.** `get-locations-rooms` (`GET /v2/locations/rooms`) and
   `get-v2-locations-spaces` (`GET /v2/locations/spaces`) give you the ids that every
   occupancy reading refers to. `get-v2-locations-room-sensors` tells you which rooms
   actually have a sensor — a room with no sensor will never report.
2. **Read the current value** at the grain the question needs. Do not sum seat occupancy to
   get room occupancy; they are independently calculated.
3. **Read history for trend questions** using the matching `*-history` operation.
4. **Render the floor plan** with `get-v3-locations-images`
   (`GET /v3/locations/images/{type}/{id}`) if you need a visual.

## Rules that will bite you

- **All three history endpoints are in the 30 requests/minute bucket.** Batch your queries
  over a wide time range rather than looping per room.
- **The sensor occupancy history endpoint no longer exists.** It was removed from the
  documentation in the 2025-02 release because it stopped returning data. Do not code against
  it.
- Occupancy history limits are **configurable per company** by subscription plan, so the
  30/minute figure is a default, not a guarantee.
- For a live feed instead of polling, use the Occupancy stream subscription — see
  `kontaktio-configure-realtime-stream`.

## Related artifacts

`asyncapi/kontaktio-streams-events.yml` · `rate-limits/kontaktio-rate-limits.yml`
