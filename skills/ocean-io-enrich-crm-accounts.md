---
name: Enrich CRM accounts in batch
description: >-
  Fill missing firmographics across a list of CRM company domains — warm the domains first, submit a
  batch enrichment keyed by your own CRM IDs, then reconcile the webhook payload back onto the right
  records, including retrying the domains Ocean.io had to crawl.
api: openapi/ocean-io-api-openapi.yml
operations:
  - warmupCompanies
  - enrichCompanies
  - enrichCompany
  - lookupCompanies
generated: '2026-08-13'
method: generated
source: >-
  openapi/ocean-io-api-openapi.yml (operationIds verified verbatim) +
  https://app.ocean.io/docs/getting-started/workflows
---

# Enrich CRM accounts in batch

Base URL `https://api.ocean.io`. Auth is the `X-Api-Token` header.

## Cost model

Batch enrichment costs **0.1 credits per result**; 1,000 companies is 100 credits.
`warmupCompanies` is **free**. If you only need identity resolution rather than full firmographics,
`lookupCompanies` is 0.05 credits per result — half the price of enrich.

## Step 1 — Warm the domains (free, do it first)

`warmupCompanies` — `POST /v2/warmup/companies` with `{"domains": [...]}` (up to 500 per call).

The response splits your input into `successfulDomains` (already indexed, enrich now) and
`triggeredDomains` (not in the database — crawling has started, wait 2–5 minutes). Doing this first
means you do not burn enrichment credits discovering that a domain was not there.

## Step 2 — Submit the batch

`enrichCompanies` — `POST /v2/enrich/companies`, up to 10,000 domains per call.

```
{
  "companyDataMapping": {
    "crm-account-001": { "company": { "domain": "company1.com" } },
    "crm-account-002": { "company": { "domain": "company2.com" } }
  },
  "webhookUrl": "https://yourapp.com/webhooks/ocean-enrich"
}
```

The keys of `companyDataMapping` are **your** record IDs and they come back in the webhook payload —
that is the reconciliation contract. Use the CRM primary key, not the domain, so a domain change
does not orphan the row.

The call returns `200` with `{"status": "in progress"}`. That is an acknowledgement, not a result.

For a single record use `enrichCompany` (`POST /v2/enrich/company`) instead — note it answers `201`,
not `200`, when the domain was not in the database and crawling was triggered.

## Step 3 — Handle the webhook

Payload: `{"results": {"<your-id>": {"status": ..., "company": ...}}}`.

| `status` | Meaning | Action |
|---|---|---|
| `found` | Matched and enriched | Write `industries`, `employeeCountOcean`, `revenue`, `technologies` back to the record |
| `not_found` | No match in the database | Mark the record; do not retry blindly |
| `triggered` | Not in the database, crawl started | Re-enrich in 2–5 minutes; cap at ~3 attempts |

Receiver rules:

- Return `2xx` immediately, process on a queue. A slow handler is read as a failure and retried.
- Deduplicate on your own record IDs — Ocean.io publishes no idempotency key on the request side and
  the same payload may be delivered more than once.
- No webhook signature exists. Embed a secret in the `webhookUrl` and validate it, or request the
  sending IP range from support and allowlist it.

Timing: 1–5 minutes for a small batch, 10–30 minutes for 10,000 records.

## Headcount fields — pick deliberately

- `companySizes` — Ocean.io bracket estimate. Use for coarse segmentation.
- `employeeCountOcean` — Ocean.io numeric estimate. Use for precise thresholds.
- `employeeCountLinkedin` — LinkedIn self-reported. Lags, and contractors inflate it.

## Errors

`402` means the credit pool is exhausted — the request was not processed and nothing was charged.
`422` means a field failed validation; read `detail[]` and check enums against
`getDataFieldsPublic`. Full catalogue: `errors/ocean-io-problem-types.yml`.
