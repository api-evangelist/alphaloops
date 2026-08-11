---
name: alphaloops-carrier-vetting
description: >-
  Vet a motor carrier before booking a load or writing a policy — resolve the carrier from a name
  or DOT number, confirm operating authority and insurance are active, read the safety record, and
  check fraud/chameleon risk signals. Use when asked to verify, vet, qualify, or run diligence on a
  trucking company or carrier.
generated: '2026-08-11'
method: generated
source: >-
  Grounded in the live OpenAPI 3.1 at openapi/alphaloops-fmcsa-carrier-data-api-openapi.json.
  Every operationId below is present in that spec. Flow mirrors the provider's own published
  "Carrier Vetting Checklist" guide and the safety due-diligence example in its CLI AGENTS.md.
api: AlphaLoops FMCSA Carrier Data API
base_url: https://api.runalphaloops.com
auth: 'Authorization: Bearer <api_key>'
operations:
  - searchCarriers
  - getCarrierByDot
  - getCarrierByMc
  - getCarrierAuthority
  - getCarrierInsurance
  - getCarrierInspections
  - getCarrierCrashes
  - getCarrierRiskSignals
  - getCarrierConnections
  - getCarrierMcSales
---

# Carrier vetting

Verify a carrier is real, authorized, insured, and not a fraud risk before doing business with it.

## 1. Resolve the carrier to a DOT number

If you were given a name, resolve it first — never assume a DOT number.

- `searchCarriers` — `GET /v1/carriers/search`
  - `company_name` is **required**. Pass `domain`, `state`, `city` when you have them; the docs
    state domain improves match accuracy.
  - `limit` maxes out at **50** here (default 10). Pagination is `page`/`limit`.
  - Read `results[].confidence` and `pagination.total_results`. A confidence well below 1.0 or
    several near-equal candidates means **stop and disambiguate** — do not silently take
    `results[0]`. Multiple legal entities share a brand name in this dataset by design.

If you were given an MC/MX docket number instead, use `getCarrierByMc`
(`GET /v1/carriers/mc/{mc_number}`) and take `dot_number` from the response.

## 2. Pull the profile

- `getCarrierByDot` — `GET /v1/carriers/{dot_number}`
  - Use `?fields=` to project only what you need. The full profile is 200+ fields, and field
    projection is supported **only** on this operation and `getCarrierByMc`.
  - For vetting, a good projection is:
    `legal_name,dba_name,mc_number,operating_authority_status,authority_types,safety_rating,power_units,drivers,physical_address,date_added`
  - `date_added` matters: a very recent registration paired with a large fleet is a red flag.

## 3. Confirm authority and insurance are active

- `getCarrierAuthority` — `GET /v1/carriers/{dot_number}/authority`
  - Returns grants, revocations and reinstatements. Uses **offset/limit** pagination, not
    page/limit. A revocation followed by a reinstatement is normal; repeated cycles are not.
- `getCarrierInsurance` — `GET /v1/carriers/{dot_number}/insurance`
  (or `getCarrierInsuranceByMc` when working from an MC number)
  - Confirm a policy is currently on file. `operating_authority_status: AUTHORIZED` on the profile
    is not by itself proof of active coverage — check the filing.

## 4. Read the safety record

- `getCarrierInspections` — `GET /v1/carriers/{dot_number}/inspections` (**offset/limit**)
- `getInspectionViolations` — `GET /v1/inspections/{inspection_id}/violations` for any inspection
  worth drilling into
- `getCarrierCrashes` — `GET /v1/carriers/{dot_number}/crashes` (**page/limit**)
  - Severity values are `FATAL`, `INJURY`, `TOW`, `PROPERTY_DAMAGE`.

**Zero inspections is not a clean record.** A carrier with authority but no inspection history has
not been observed operating — treat it as unknown, not good.

## 5. Check fraud and chameleon risk

- `getCarrierRiskSignals` — `GET /v1/carriers/{dot_number}/risk-signals`
  - Consolidated risk scoring with triggered combinations.
- `getCarrierConnections` — `GET /v1/carriers/{dot_number}/connections`
  - The corporate-connection graph — entities linked by shared VINs, officers, phone numbers and
    addresses. This is what surfaces a chameleon carrier reincarnated under a new DOT number.
- `getCarrierMcSales` — `GET /v1/carriers/{dot_number}/mc-sales`
  - Whether the authority is being shopped for sale. An aged MC number with original officers
    still on file but new operational control is the specific pattern the provider's own May 2026
    risk-scoring rebuild was aimed at.

## Runtime rules

- **Auth**: `Authorization: Bearer <key>`. A 401 means the key is missing or invalid — it will not
  fix itself on retry, so do not retry.
- **Rate limits**: 60 requests/minute and 5,000/day on the Enterprise REST tier. Both windows are
  reported on *every* response via `X-RateLimit-Limit/Remaining/Reset` and
  `X-DailyLimit-Limit/Remaining/Reset`. Pace off those headers rather than counting your own calls.
  On 429, honour `Retry-After` (seconds).
- **Pagination is not uniform**: `authority`, `inspections`, `trucks` and `trailers` use
  `offset`/`limit`; everything else uses `page`/`limit`. Check before you loop.
- **404 vs empty**: a 404 means the carrier does not exist. A sub-resource with no data returns
  **200 with an empty results array** — do not report "carrier not found" for an empty fleet list.
- **Errors** are always `{"error": "...", "message": "..."}`. There is no published enumeration of
  `error` values, so branch on the HTTP status, not the string.
- 500 and 502 are retryable with backoff; 502 specifically means an upstream data provider failed.

## What to report

Lead with the decision, then the evidence: legal name and DOT, authority status, insurance on
file yes/no, fleet size, safety rating, crash and inspection counts, and every triggered risk
signal. Say explicitly when a check could not be completed — an unavailable insurance filing is a
material gap, not a pass.
