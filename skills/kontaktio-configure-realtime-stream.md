---
name: kontaktio-configure-realtime-stream
description: Stand up a Kio Cloud Streams subscription so telemetry, position or occupancy events are pushed into your own AWS Kinesis, Azure Event Hub or IBM Watson service, and tear it down safely.
api: Kontakt.io Location & Occupancy API
spec: openapi/kontaktio-location-occupancy-openapi.yml
base_url: https://apps.cloud.us.kontakt.io
operations:
  - post-v3-channels
  - get-v3-channels
  - get-v3-channels-id
  - patch-v3-channels-id
  - get-v3-channels-state
  - delete-v3-channels-id
  - post-v3-streams-subscriptions-position
  - get-v3-streams-subscriptions-position
  - patch-v3-streams-subscriptions-position
  - delete-v3-streams-subscriptions-position
  - post-v3-streams-subscriptions-telemetry
  - get-v3-streams-subscriptions-telemetry
  - patch-v3-streams-subscriptions-telemetry
  - delete-v3-streams-subscriptions-telemetry
  - post-v3-streams-subscriptions-type-channels
  - delete-v3-streams-subscriptions-type-channels
---

# Configure a real-time stream

Two objects: a **channel** is a third-party streaming service you own; a **subscription** is
the class of Kio Cloud data pushed to it. There are three subscription types — Telemetry,
Position, Occupancy.

**This is not a webhook.** Kio Cloud will not POST to a URL you supply. You must already own
a Kinesis stream, an Event Hub or a Watson instance.

## Steps

1. **Create the channel.** `post-v3-channels` (`POST /v3/channels`) with the payload shape for
   your provider (`Stream-channel-kinesis`, `Stream-channel-eventHub` or
   `Stream-channel-watson`). The response carries the channel id you will need next.
2. **Optionally filter at the channel.** Set `deviceModels` (array of device model ids) and/or
   `campusIds`. No filters means the channel receives every event on the subscription. Both
   filters means an event must match device model **and** campus.
3. **Create the subscription** for the data type you want, e.g.
   `post-v3-streams-subscriptions-position` (`POST /v3/streams/subscriptions/position`),
   referencing the channel id in `channelIds`.
4. **Add or remove channels later without rewriting the list.** Use
   `post-v3-streams-subscriptions-type-channels` and
   `delete-v3-streams-subscriptions-type-channels` (added 2025-06). A `PATCH` on the
   subscription overwrites `channelIds` wholesale — that is the operational risk these two
   endpoints exist to remove.
5. **Verify delivery.** `get-v3-channels-state` (`GET /v3/channels/state/{id}`) returns the
   channel connection state. Check it before assuming silence means "no events".
6. **Tear down in order:** delete the subscription first
   (`delete-v3-streams-subscriptions-position`), then the channel (`delete-v3-channels-id`).

## Rules that will bite you

- **One subscription per type per company.** You cannot create a second Position subscription;
  the create call is not a way to add a parallel feed. If one exists, PATCH it or add channels
  to it.
- **Five concurrent stream connections** per account. That is a connection-based limit, not a
  request-based one.
- **The occupancy subscription operations reuse the telemetry operationIds** in the published
  spec (`post-v3-streams-subscriptions-telemetry` etc. appear on both paths). Bind to the
  **path**, not the operationId, or your generated client will collide.
- Every step here is reversible — see the `reversibility:` block in
  `conventions/kontaktio-conventions.yml` — but Kontakt.io publishes no window for any of it.

## Related artifacts

`asyncapi/kontaktio-streams-events.yml` · `conventions/kontaktio-conventions.yml`
