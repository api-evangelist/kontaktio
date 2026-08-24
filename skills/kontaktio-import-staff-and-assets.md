---
name: kontaktio-import-staff-and-assets
description: Batch import or update Staff and Asset entities into the Kontakt.io Kio Apps platform from a third-party system of record, using the Entity Management Integration API.
api: Kontakt.io Entity Management Integration API
spec: openapi/kontaktio-entity-management-openapi.yml
base_url: https://api.cloud.us.kontakt.io/entity-management
operations:
  - createUpdateEntities
  - entityDetails
---

# Import staff and assets

This is the integration seam between an HR/CMMS/EMR system of record and Kio Cloud.

## Before you start — this API does NOT take an Api-Key

It is the one Kontakt.io surface with a fully documented OAuth2 flow, and it needs setup in
the Kio Cloud UI first:

1. A **User Management Administrator** assigns the `integration-api` role to a user
   (Users > select user > Roles > Edit > User Management > integration-api).
2. That user creates an **Integration API Client** (Users > Integration API > + Add Client).
   The Client ID is prefixed `api-{companyId}-`.
3. Assign the client the `integration-api` **client role** and copy the **Client secret**.

Then mint a token:

```
POST https://kc.cloud.{region}.kontakt.io/realms/{tenant}/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=...&client_secret=...&scope=profile email openid&grant_type=client_credentials
```

`{region}` is `us` or `uk`; `{tenant}` is the Tenant Name from your account URL
`https://{tenant}.app.cloud.{region}.kontakt.io`.

## Steps

1. **Cache the token.** It expires in **300 seconds**. Kontakt.io explicitly warns that
   requesting a token per API call may get you rate limited. Renew ~30 seconds before expiry.
2. **Push the batch.** `createUpdateEntities` (`POST /v1/integration/entity`) with
   `Authorization: Bearer <token>`. It creates or updates in one call.
3. **Verify a record.** `entityDetails` (`GET /v1/integration/entity/{entityId}`) returns the
   current configuration of one entity.
4. **On 401, refresh once and retry once.** Do not loop.

## Rules that will bite you

- **There is no delete and no revert.** The whole API is two operations. A bad batch cannot be
  rolled back through this contract — reconcile against your source system and re-import a
  corrected batch instead. This is the one genuinely irreversible write in the Kontakt.io
  estate; see `conventions/kontaktio-conventions.yml`.
- **There is no idempotency key.** A retried batch will be reapplied. Because the operation is
  create-or-update keyed on your entity ids, a duplicate submission of the *same* payload is
  usually harmless, but a partially-applied batch followed by a retry is not deduplicated.
- Errors are `{message, errors[{field, error}]}`; a 400 lists every failing field, so read all
  of `errors[]` rather than the first entry.

## Related artifacts

`authentication/kontaktio-authentication.yml` · `scopes/kontaktio-scopes.yml` ·
`errors/kontaktio-problem-types.yml`
