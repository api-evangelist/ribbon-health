---
name: ribbon-health-directory-maintenance
description: >-
  Maintain a customer's custom provider directory on the H1 (Ribbon Health) API — create and edit
  locations, attach specialties, and set which insurances a provider accepts at which location.
  This is the write surface; it has no idempotency key, so read the retry rules first.
api: openapi/ribbon-health-providers-api-openapi.yml
operations:
  - postCustomLocations
  - getCustomLocation
  - putCustomProvider
  - putCustomProviderLocations
  - putCustomProviderSpecialties
  - putCustomProviderPrimarySpecialties
  - putCustomProviderLocationInsurances
  - putCustomLocationInsurances
  - getCustomProvider
generated: '2026-08-14'
method: generated
source: >-
  openapi/ribbon-health-providers-api-openapi.yml,
  openapi/ribbon-health-locations-api-openapi.yml,
  https://ribbon.readme.io/llms.txt,
  conventions/ribbon-health-conventions.yml
---

# Directory maintenance (H1 / Ribbon Health API)

Use this to keep a payer's or digital-health company's provider directory accurate on top of
H1's base data. Everything under `/v1/custom/*` is the customer's overlay on H1's records.

**Base URL:** `https://api.ribbonhealth.com/v1`
**Auth:** `Authorization: Bearer {customer_token}`

## Retry rules — read first

**This API documents no idempotency key.** There is no `Idempotency-Key` header anywhere in the
spec or the docs.

- **`PUT` operations are safe to retry.** The whole relationship-editing surface
  (`putCustomProviderLocations`, `putCustomProviderSpecialties`,
  `putCustomProviderLocationInsurances`, `putCustomLocationInsurances`) is a PUT and is naturally
  idempotent — replaying it converges on the same state.
- **`POST` creates are NOT safe to retry.** `postCustomLocations` and its siblings will create a
  duplicate record on a retry after a timeout. The only guard is field-uniqueness: some create
  paths return `409` with "object with given fields already exists". That is uniqueness
  enforcement, not idempotency — do not rely on it.
- **After a timeout on a POST, read before you retry.** Search for the record you were creating
  (`getCustomLocations` with the same address/name) and only re-POST if it is genuinely absent.

## Steps

1. **Create the location if it does not exist** — `postCustomLocations`
   (`POST /custom/locations`). Use this for urgent care sites, labs, imaging or therapy centres
   not already in H1's data. Confirm with `getCustomLocation`
   (`GET /custom/locations/{location_uuid}`).

2. **Edit provider fields** — `putCustomProvider` (`PUT /custom/providers/{npi}`). Edits
   everything that is *not* `specialties`, `locations` or `insurances` — those have dedicated
   operations. You may add new custom fields or remove existing ones here.

3. **Attach the provider to locations** — `putCustomProviderLocations`
   (`PUT /custom/providers/{npi}/locations`), using H1's standard location UUIDs.

4. **Set specialties** — `putCustomProviderSpecialties`
   (`PUT /custom/providers/{npi}/specialties`) with standard specialty UUIDs, then
   `putCustomProviderPrimarySpecialties`
   (`PUT /custom/providers/{npi}/specialties/{specialty_uuid}`) to flag the primary ones.

5. **Set network participation — per location** — `putCustomProviderLocationInsurances`
   (`PUT /custom/providers/{npi}/locations/{location_uuid}/insurances`).

   This is the operation that matters most and the one most integrations get wrong. **Insurance
   acceptance is an edge on the (provider, location) pair, not a property of the provider.** A
   provider can be in-network at their private practice and out-of-network at the hospital clinic
   where they also see patients. Writing insurances at the provider level (or reading the
   provider's top-level `insurances` union and presenting it as "where they take your plan")
   produces exactly the directory inaccuracy this product exists to fix.

   Use `putCustomLocationInsurances` (`PUT /custom/locations/{location_uuid}/insurances`) when the
   fact is about the facility rather than the individual.

6. **Verify** — `getCustomProvider` (`GET /custom/providers/{npi}`) and read back the merged view.

## Latency

Edit operations accept `async=true`, which applies the edit asynchronously and returns faster.
Note there is **no job id, no status endpoint and no completion webhook** — you have no published
way to confirm an async edit landed. If you need confirmation, either write synchronously or
poll the resource with `getCustomProvider`.

## Destructive operations

`deleteCustomInsurance` deletes **every instance of that UUID** across all your providers, and H1
warns it cannot regenerate them. Same shape for `deleteCustomSpecialty`, `deleteCustomLocation`,
`deleteCustomProviderType`, `deleteCustomLocationType`. Treat all deletes as
`human-in-the-loop: required` — see `agentic-access/ribbon-health-agentic-access.yml`.

You also cannot edit or delete H1-created specialties, provider types or location types, only
customer-created ones; attempting it fails.

## Errors

`{"error": {"status": <int>, "code": "<snake_case>", "message": "<string>"}}`. On this surface:
`403` means the customer's package excludes the capability (the spec's own example is "Trial
accounts do not have access to custom specialties"); `404` means a referenced UUID does not
exist — re-resolve it through the reference endpoints; `409` means a uniqueness conflict on a
create. See `errors/ribbon-health-problem-types.yml`.
