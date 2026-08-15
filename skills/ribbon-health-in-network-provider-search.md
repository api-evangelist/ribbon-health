---
name: ribbon-health-in-network-provider-search
description: >-
  Find in-network providers near a member using the H1 (Ribbon Health) API — resolve the
  insurance carrier and specialty to UUIDs first, then search the provider directory, then read
  the full provider record.
api: openapi/ribbon-health-providers-api-openapi.yml
operations:
  - getInsurances
  - getSpecialties
  - getCustomProviders
  - getCustomProvider
generated: '2026-08-14'
method: generated
source: >-
  openapi/ribbon-health-providers-api-openapi.yml,
  openapi/ribbon-health-reference-endpoints-api-openapi.yml,
  https://ribbon.readme.io/docs/search-networks,
  conventions/ribbon-health-conventions.yml
---

# In-network provider search (H1 / Ribbon Health API)

Use this when a user asks "find me a cardiologist near me who takes my insurance".

**Base URL:** `https://api.ribbonhealth.com/v1`
**Auth:** `Authorization: Bearer {customer_token}` on every request. A missing key returns
`401 not_authenticated`; a bad key returns `401 authentication_failed`. Branch on
`error.code`, never on `error.message` — the same code ships two different messages.

## Never skip the resolve step

Insurance and specialty are **UUID-keyed**. You cannot search by the string "Aetna" or
"Cardiology". Resolve first, every time.

## Steps

1. **Resolve the insurance to a UUID** — `getInsurances` (`GET /custom/insurances`).
   Search by carrier/plan name. This endpoint returns the **reference envelope**:
   `{count, next, previous, results}`. Read `results`, not `data`.

2. **Resolve the specialty to a UUID** — `getSpecialties` (`GET /custom/specialties`).
   Same reference envelope: read `results`.

3. **Search the directory** — `getCustomProviders` (`GET /custom/providers`).
   Pass the resolved `insurance_ids` / `specialty_ids` plus the geography (address, or
   latitude+longitude, plus a distance). This endpoint returns the **custom-directory envelope**:
   `{parameters, data}`. Read `data`, not `results`.
   - Keep `page_size` at the default **25**. The provider's own latency guidance says raising it
     measurably slows the response, and that `specialties` and `insurances` filtering are the two
     costliest filters.
   - Use `fields` or `_excl_fields` to trim the payload — the number of returned fields is a
     documented latency driver.

4. **Read the full record** — `getCustomProvider` (`GET /custom/providers/{npi}`) for the
   provider the user picks. The key is the federal **NPI**, so you can hold it across sessions
   and join it to any other NPI-keyed dataset.

## Two things that will bite you

- **Three list envelopes exist in this one API.** `/custom/providers` returns
  `{parameters, data}`; `/custom/insurances` and `/custom/specialties` return
  `{count, next, previous, results}`; the `/v2/*` pricing endpoints return
  `{parameters, total_count, page, page_size, data}`. A shared paginator written against one
  will silently read zero rows from another. See `conventions/ribbon-health-conventions.yml`.

- **Network status is per (provider, location), not per provider.** A provider's `insurances`
  array is the union across all their practice locations. If the user needs to know where to
  actually go, read the location-level insurances rather than trusting the top-level array. See
  `data-model/ribbon-health-data-model.yml`.

## Data quality

Records carry a `confidence` score. The product H1 sells is provider-data *accuracy* — threshold
on confidence rather than presenting every row as equally true. See
https://ribbon.readme.io/docs/confidence-scores.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 `not_authenticated` | No key sent | Add the bearer header |
| 401 `authentication_failed` | Key rejected | Stop; the credential is wrong |
| 400 | Malformed request | Usually a missing or conflicting query parameter |
| 403 | Entitlement, not auth | The customer's package excludes this module. Escalate to a human — retrying will not help |
| 404 `not_found` | Unknown route or unknown UUID/NPI | Re-resolve the UUID |
| 429 `rate_limit_exceeded` | Over the documented limit | Back off. **No `Retry-After` or `RateLimit-*` header is returned** — you must implement your own backoff from the documented 1,000/min (search) or 5,000/min (single lookup) |

Quote the `ribbon-request-id` response header to support. It is returned on every response,
including errors, and is undocumented.
