---
name: ribbon-health-cost-and-eligibility
description: >-
  Estimate what a procedure will cost a specific member on the H1 (Ribbon Health) API — check the
  member's live benefits, then look up insurance-specific negotiated rates by provider and
  location. Handles PHI; read the safety rules before automating any of it.
api: openapi/ribbon-health-cost-estimates-api-openapi.yml
operations:
  - getEligibilityInsurancePartners
  - getEligibility
  - getProcedureCostEstimate
  - getPricingProviders
  - getPricingProviderProcedures
  - getPricingProviderProcedure
generated: '2026-08-14'
method: generated
source: >-
  openapi/ribbon-health-cost-estimates-api-openapi.yml,
  openapi/ribbon-health-price-transparency-api-openapi.yml,
  https://ribbon.readme.io/docs/eligibility-check,
  https://ribbon.readme.io/docs/cost-quality,
  conventions/ribbon-health-conventions.yml
---

# Cost and eligibility (H1 / Ribbon Health API)

Use this when a user asks "what will this cost me?" — which is two different questions the API
answers with two different products.

**Base URL:** `https://api.ribbonhealth.com/v1`
**Auth:** `Authorization: Bearer {customer_token}`

## PHI warning — read before automating

`getEligibility` (`POST /eligibility`) is the only operation here that transmits **protected
health information**: member identifiers, plan details, deductible and out-of-pocket progress,
and per-service copay and coinsurance summaries. Treat it as `human-in-the-loop` by default:

- Never call it speculatively or to "warm a cache".
- Never log the request or the response body.
- Never surface another person's benefits to the caller.
- H1 publishes no trust center and no named certification (SOC 2 / HITRUST / HIPAA attestation)
  on any public page — see `conformance/ribbon-health-conformance.yml`. Confirm your own BAA and
  contractual position before wiring this into an autonomous agent.

See `agentic-access/ribbon-health-agentic-access.yml`.

## Path A — what does this member owe? (benefits)

1. **Check the carrier is supported** — `getEligibilityInsurancePartners`
   (`GET /eligibility_insurance_partners`). Not every carrier is covered by the eligibility
   product; check before you promise the user an answer.
2. **Check benefits** — `getEligibility` (`POST /eligibility`). Returns `plan_info`,
   `deductible_detail`, `out_of_pocket_detail` and per-service summaries
   (`primary_care_summary`, `specialist_office_summary`, `surgical_summary`, `mri_ct_scan_summary`,
   `urgent_care_summary`, and a dozen more). The response carries its own `request_id`.
3. **Estimate the procedure** — `getProcedureCostEstimate` (`GET /procedure_cost_estimate`)
   combines the procedure and the member's location into an estimate.

## Path B — what do providers charge? (negotiated rates)

1. **Shop by procedure** — `getPricingProviders` (`GET /pricing/providers`): providers who
   perform a procedure, with the lowest insurance-specific negotiated rate near an address.
2. **All procedures for one provider** — `getPricingProviderProcedures`
   (`GET /pricing/providers/{npi}/procedures`).
3. **One procedure, one provider, across their locations** — `getPricingProviderProcedure`
   (`GET /pricing/providers/{npi}/procedures/{procedure_uuid}`), and the fully-qualified
   `getPricingProviderProcedureLocation` when you already know the location UUID.

A price is only meaningful at the intersection of **provider × procedure × insurance ×
location**. Quoting a rate without all four is quoting a number that does not apply to the user.

## The v1 / v2 trap

There is a newer **Price Transparency v2** surface at `https://api.ribbonhealth.com/v2` —
`/v2/procedures`, `/v2/carriers`, `/v2/care-clusters`, `/v2/pricing/locations/procedures`,
`/v2/pricing/locations/care-clusters`. It is documented by the provider and confirmed live, but
**no OpenAPI in this repo covers it**, so the operations above are all v1. If you move to v2:

- **v2 carrier ids are a different identifier space from v1 `/pricing/carriers` UUIDs.** Do not
  carry one across.
- **v2 requires an explicit location** (`address`, or both `lat` and `lng`). v1 silently
  defaulted to a New York City address; v2 returns `400` instead. That silent v1 default is worth
  knowing about on its own — a v1 pricing search with no address has been answering about
  Manhattan.
- `plan_id` parses but returns `501`. Use `carrier_id`.
- `carrier_id` and `plan_id` together return `400`.

Access to the whole pricing surface is gated by the `doctors.can_price_transparency` entitlement;
without it you get `403`, which no retry will fix.

## Errors and limits

`{"error": {"status": <int>, "code": "<snake_case>", "message": "<string>"}}` — not RFC 9457.
Eligibility is the strictest limit in the API: **1,000 requests per hour**, not per minute.
No `RateLimit-*` or `Retry-After` header is returned, so implement backoff yourself. See
`errors/ribbon-health-problem-types.yml` and `rate-limits/ribbon-health-rate-limits.yml`.
