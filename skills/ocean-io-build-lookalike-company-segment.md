---
name: Build a lookalike company segment from seed customers
description: >-
  Turn three to five closed-won customer domains into a ranked list of similar companies you can use
  as an outbound segment, paginating the full result set with the searchAfter cursor.
api: openapi/ocean-io-api-openapi.yml
operations:
  - searchCompaniesV3
  - previewSearchCompaniesV3
  - getCreditBalance
generated: '2026-08-13'
method: generated
source: >-
  openapi/ocean-io-api-openapi.yml (operationIds verified verbatim) +
  https://app.ocean.io/docs/getting-started/workflows
---

# Build a lookalike company segment from seed customers

Base URL `https://api.ocean.io`. Auth is the `X-Api-Token` header.

## Cost model

`searchCompaniesV3` costs **0.2 credits per result**. 500 companies is 100 credits. Lookalike search
costs the same per result as filter search on v3, so the price of a bad seed set is wasted credits,
not a surcharge — qualify first.

`previewSearchCompaniesV3` exists but the credits documentation records preview endpoints as "not
available on this plan" for the self-serve credit plan. Do not assume you can call it.

## Step 1 — Check the balance

`getCreditBalance` — `GET /v2/credits/balance`. Read `credits.recurrent` and `dailyLimitRateLeft`
before committing to a large run.

## Step 2 — Probe the result count

`searchCompaniesV3` — `POST /v3/search/companies` with `"size": 1`. Read `total` from the response.
If `total` is far off what you expect, fix the filters before spending.

## Step 3 — Run the lookalike search

`searchCompaniesV3` — `POST /v3/search/companies`

In `companiesFilters`:

- `lookalikeDomains` — 3–5 of your best-fit customers. Diverse seeds broaden the result set; similar
  seeds narrow it.
- `excludeDomains` — repeat the seed domains here so your own customers do not come back as results.
- `companySizes`, `primaryLocations.includeCountries` (lowercase alpha-2), `industries` — narrow the
  net. Every enum must match `getDataFieldsPublic` (`GET /v2/data-fields`).
- `minScore` — the similarity floor. If a lookalike search returns zero rows, lowering or removing
  `minScore` is the first thing to try.

Use `fields` to request only the attributes you will actually store.

## Step 4 — Paginate

Take `searchAfter` from the response and send it back on the next request with the identical filter
block. Repeat until `searchAfter` is absent or null. `size` maxes at 10,000 per page for company
search. Cursor pagination means no duplicates and no skips even while the index changes underneath
you.

## Step 5 — Qualify

Each company row carries `domain`, `name`, `companySize`, `industries`, `technologies`, `revenue`
and growth signals — enough to score the segment before any outreach. `domain` is the primary key
across the whole API; keep it as your join key.

## Related

- Cluster an existing customer list instead of expanding one:
  `ocean-io-segment-and-attribute-accounts.md`
- Cheaper path when you already have the domains: `lookupCompanies` costs 0.05 credits per result
  versus 0.2 for search with `includeDomains`.
