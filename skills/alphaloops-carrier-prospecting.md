---
name: alphaloops-carrier-prospecting
description: >-
  Build a targeted list of motor carriers and find the decision-makers to contact — filter the
  2.7M-carrier universe by geography, fleet size, equipment and technology stack, then find and
  enrich contacts at the matching companies. Use when asked to build a prospect list, find carriers
  matching criteria, find lookalikes of existing customers, or get contact details at a carrier.
generated: '2026-08-11'
method: generated
source: >-
  Grounded in the live OpenAPI 3.1 at openapi/alphaloops-fmcsa-carrier-data-api-openapi.json.
  Every operationId below is present in that spec. Flow mirrors the provider's published
  "How to Prospect With FMCSA Data" guide and the contact examples in its CLI AGENTS.md.
api: AlphaLoops FMCSA Carrier Data API
base_url: https://api.runalphaloops.com
auth: 'Authorization: Bearer <api_key>'
operations:
  - queryCarriers
  - searchCarriers
  - getSimilarCarriers
  - getCarrierByDot
  - getCarrierOverview
  - searchContacts
  - enrichContact
  - getCarrierTrucks
  - getCarrierTrailers
  - getCarrierTimeline
---

# Carrier prospecting

Go from a target profile to a contactable list.

## 1. Choose the right entry point

Three different starting points, three different operations:

| You have | Use | Why |
|---|---|---|
| A set of filter criteria | `queryCarriers` | The advanced filter — the only operation that can build a list from nothing |
| A company name | `searchCarriers` | Fuzzy name match with a confidence score |
| An existing good customer (DOT) | `getSimilarCarriers` | Lookalikes, ranked by the provider's embedding model |

## 2. Build the list with the advanced filter

- `queryCarriers` — `POST /v1/carriers/query`

This is the most capable operation in the API and the one worth learning. The request body takes
`include` and `exclude` objects plus projection and sorting:

```json
{
  "include": {
    "state": "TX",
    "power_units": { "min": 50 },
    "cargo_type": ["Van", "Reefer"],
    "operating_authority_status": "Active",
    "location": { "latitude": 29.76, "longitude": -95.37, "radius_miles": 100 }
  },
  "exclude": { "safety_rating": "Unsatisfactory" },
  "fields": ["dot_number", "legal_name", "power_units", "state"],
  "sort_by": "power_units",
  "sort_order": "desc",
  "limit": 25,
  "page": 1
}
```

- Filter values can be scalars, range objects (`{"min": …}`), arrays, or a geo-radius block.
- `sort_by` accepts `power_units`, `drivers`, `date_added`, `safety_rating`, `annual_revenue`,
  `distance`; `sort_order` is `asc` or `desc` (default `desc`).
- Always pass `fields` — the default profile is 200+ fields and you are paying latency for all of
  them.
- Page with `page`/`limit` until `page >= pagination.total_pages`. Default `limit` here is 25.

## 3. Or expand from a known-good customer

- `getSimilarCarriers` — `GET /v1/carriers/{dot_number}/similar`
  - Feed it the DOT of a customer that closed well. Ranked by similarity across fleet profile.
- `getCarrierOverview` — `GET /v1/carriers/{dot_number}/overview` for a fast summary of any
  candidate before spending a full profile call.

## 4. Qualify the candidates

- `getCarrierByDot` with `?fields=` for the firmographics you actually score on.
- `getCarrierTrucks` / `getCarrierTrailers` — `GET /v1/carriers/{dot_number}/trucks|trailers`
  (**offset/limit** pagination) when equipment mix matters: make, model year, GVW, cab type,
  reefer flag. This is VIN-level, not registration paperwork.
- `getCarrierTimeline` — `GET /v1/carriers/{dot_number}/timeline` (**offset/limit**) for change
  events with `category`, `event_type`, `field_name`, `old_value`, `new_value`. Filter by
  `category` (contact, address, fleet, operations, authority, people) and `date_from`/`date_to`.
  A carrier whose `power_units` jumped last quarter is in a different buying moment than one flat
  for three years. This is also the closest thing the API has to an event feed — poll it.

## 5. Find and enrich decision-makers

- `searchContacts` — `GET /v1/contacts/search`
  - Filter by `dot`, `company`, `title`, and `levels` (`c_suite`, `vp`, `director`, `manager`).
  - **This operation can return `202 Accepted`** — contacts are being fetched asynchronously.
    That is a success, not an error. Re-issue the same request after a delay. No job id or poll
    interval is published, so back off progressively (e.g. 2s, 5s, 15s) and give up gracefully.
  - The response array is keyed `contacts`, not `results`.
- `enrichContact` — `GET /v1/contacts/{contact_id}/enrich`
  - Adds work email, phone numbers, location and employment history.

### Enrichment costs real money — treat it as metered

- **1 credit per NEW enrichment. Cached results are free.**
- Remaining balance comes back two ways: the `credits` object in the body
  (`{used, remaining, total}`) and the `X-Enrichment-Credits-Remaining` header.
- **402 Payment Required** means credits are exhausted. Do not retry — it will not recover.
- Never enrich a whole list speculatively. Qualify first, enrich only the contacts you will
  actually use, and check `remaining` before starting a batch.

## Runtime rules

- **Auth**: `Authorization: Bearer <key>`. 401 is terminal, do not retry.
- **Rate limits**: 60/minute, 5,000/day on the Enterprise REST tier. Read
  `X-RateLimit-Remaining` and `X-DailyLimit-Remaining` off every response and pace against the
  tighter of the two. On 429, honour `Retry-After`.
- **Pagination is not uniform**: `trucks`, `trailers`, `inspections`, `authority` and `timeline`
  use `offset`/`limit`; search and contacts use `page`/`limit`. Result arrays are variously
  `results`, `contacts` or `events`.
- **Field projection** (`?fields=`) works only on `getCarrierByDot` and `getCarrierByMc`;
  `queryCarriers` takes its own `fields` array in the body. It is unsupported everywhere else.
- **Errors** are `{"error": "...", "message": "..."}`. Branch on HTTP status — the `error` string
  has no published value set.

## Privacy

Contact enrichment returns named individuals' work and personal contact details. The provider
states the contact dataset is GDPR/CCPA compliant, but lawful basis for *your* outreach is yours
to establish. Do not enrich beyond what the task needs, and do not put personal emails or mobile
numbers into any output that was not explicitly asked for.
