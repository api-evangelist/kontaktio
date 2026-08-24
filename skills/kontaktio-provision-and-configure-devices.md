---
name: kontaktio-provision-and-configure-devices
description: Claim an order, configure Kontakt.io beacons, badges and gateways, manage firmware, and grant or revoke device access using the Kio Cloud Device Management API.
api: Kontakt.io Device Management API
spec: openapi/kontaktio-device-management-openapi.yml
base_url: https://dm-api.cloud.us.kontakt.io
operations:
  - "GET /order"
  - "POST /order/claim"
  - "GET /order/claim"
  - "GET /device"
  - "POST /device/update"
  - "GET /config"
  - "POST /config/create"
  - "POST /config/delete"
  - "POST /config/export"
  - "POST /config/import"
  - "GET /firmware"
  - "POST /device/{uniqueId}/access"
  - "DELETE /device/{uniqueId}/access/{email}"
  - "POST /bulk/device-access"
  - "getBulkDeviceAccessStatusForAuthorizedManager"
  - "getBulkDeviceAccessStatus"
---

> **operationId warning.** 43 of the 45 operations in the published Device Management spec
> carry **no operationId** — only `getBulkDeviceAccessStatusForAuthorizedManager` and
> `getBulkDeviceAccessStatus` have one. The `operations:` list above therefore references
> METHOD + path, which is what the contract actually provides. Do not invent operationIds; a
> generated client will name these itself and the names will not be portable.

# Provision and configure devices

## Before you start

- Auth: `Authorization: Bearer <jwt>` from the Keycloak client-credentials flow (see
  `kontaktio-import-staff-and-assets` for how to mint one). The legacy `Api-Key` header still
  works but is marked deprecated and slated for removal — do not build new integrations on it.
- **`Accept: application/vnd.com.kontakt+json;version=10` is required.** Omitting the version
  silently pins you to "current stable"; sending an unknown version returns **415**.
- **POST bodies are `application/x-www-form-urlencoded`, not JSON.** This is the single most
  common integration mistake on this API.
- Region: `dm-api.cloud.us.kontakt.io` or `dm-api.cloud.uk.kontakt.io`.

## Steps

1. **Claim the hardware.** `GET /order` checks order ids; `POST /order/claim` claims one.
   Devices already known to the API are skipped, new ones are created. **This call is
   idempotent** — claiming the same order twice on the same account has no effect — which
   matters because there is no unclaim operation.
2. **List what you have.** `GET /device` with paging (`startIndex`, `maxResult`, `orderBy`,
   `order`). Read `searchMeta.nextResults`; an empty string means the set is exhausted.
3. **Push configuration.** `POST /config/create` creates or updates a **pending** config for
   one or more `uniqueId`s. Unsupported `customConfiguration` PIDs are **silently dropped**
   and the response body reflects what was actually accepted — diff it. A `201` with an empty
   array means none of your `uniqueId`s were accessible.
4. **Undo before it lands.** `POST /config/delete` removes a pending config. Once the device
   has applied it, deleting the pending record does not roll the hardware back, and since the
   2025-04 release some system configuration changes are **locked and cannot be cancelled at
   all**.
5. **Manage access.** `POST /device/{uniqueId}/access` grants, `DELETE
   /device/{uniqueId}/access/{email}` revokes. For fleets use `POST /bulk/device-access`
   (which performs grant *or* revoke) and poll `getBulkDeviceAccessStatus` with the job key.
6. **Firmware.** `GET /firmware` lists available versions.
   `GET /firmware/{firmwareVersion}` and `GET /firmware/{firmwareVersion}/file` are marked
   **deprecated** in the spec.

## Rules that will bite you

- **60 requests/second, account-wide.** The spec declares no 429 response, so treat any
  unexpected refusal as throttling and back off one second.
- **No idempotency key.** Apart from `POST /order/claim`, every write can be double-applied by
  a retry. Configuration writes are especially unforgiving — re-sending a create is not a
  no-op.
- **Deprecated operations still in the spec:** `GET /device/unassigned/{managerId}`,
  `GET /firmware/{firmwareVersion}`, `GET /firmware/{firmwareVersion}/file`.
  `POST /device/assign` was deprecated and hidden in 2024-05.
- 401 bodies are an opaque incident envelope — `{"id":"API_ERROR_<epoch-ms>","status":401,...}`
  — quote that id to support.

## Related artifacts

`conventions/kontaktio-conventions.yml` · `lifecycle/kontaktio-lifecycle.yml` ·
`errors/kontaktio-problem-types.yml`
