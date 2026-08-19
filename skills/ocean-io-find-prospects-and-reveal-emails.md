---
name: Find prospects at lookalike companies and reveal their emails
description: >-
  Search Ocean.io for people matching a seniority/department profile at companies that look like your
  best customer, then reveal verified email addresses for the matches. Results arrive asynchronously
  on a webhook you supply.
api: openapi/ocean-io-api-openapi.yml
operations:
  - searchPeopleV3
  - revealEmails
generated: '2026-08-13'
method: generated
source: >-
  openapi/ocean-io-api-openapi.yml (operationIds verified verbatim) +
  https://app.ocean.io/docs/getting-started/workflows
---

# Find prospects at lookalike companies and reveal their emails

Base URL `https://api.ocean.io`. Every request carries the account token in the `X-Api-Token`
header — never in both the header and the `apiToken` query parameter, which returns `400`.

## Before you start

- Credits: search costs **0.2 credits per result**, an email reveal costs **1 credit per email
  found**. Nothing is charged for `notFound`. Estimate before you run.
- Rate limits (self-serve): **60 requests/minute, 1,000 requests/day**. Check `dailyLimitRateLeft`
  from `getCreditBalance` before a large run.
- Enum values (seniorities, departments, industries) must match `getDataFieldsPublic`
  (`GET /v2/data-fields`) exactly, or the request returns `422`.
- Country codes are lowercase ISO alpha-2 — `"de"`, not `"DE"` or `"Germany"`.

## Step 1 — Size the search before spending

Call `searchPeopleV3` (`POST /v3/search/people`) once with `"size": 1` and read `total` from the
response. Decide how many results you actually want before running it for real.

## Step 2 — Search people

`searchPeopleV3` — `POST /v3/search/people`

Put the person criteria in `peopleFilters` (`seniorities`, `departments`, `jobTitleKeywords`) and
the account criteria in `companiesFilters` (`lookalikeDomains` for lookalike targeting,
`companySizes`, `industries`, `primaryLocations`). Use `fields` to request only the attributes you
need — it cuts payload and latency.

Collect the `id` value from each person in the response; that is the input to the reveal step.

Paginate with the `searchAfter` cursor from the response (see
`conventions/ocean-io-conventions.yml`). Stop when `searchAfter` is absent or null. **Do not**
combine `searchAfter` with `peoplePerCompany` — that combination is unsupported; cap with `size`
instead.

## Step 3 — Reveal emails

`revealEmails` — `POST /v2/reveal/emails`

Send `personIds` (up to 500 per call) and a `webhookUrl`. The call returns `200` with
`{"status": "in progress"}` — that is an acknowledgement, not the result.

## Step 4 — Receive the webhook

Ocean.io POSTs `{"results": [{"personId": ..., "email": {"address": ..., "status": ...}}]}` to your
URL, typically within 1–3 minutes for a small batch and 2–10 minutes for 500 IDs.

Rules your receiver must follow:

- Return `2xx` immediately and process off a queue; a slow handler is treated as a failure and
  retried with exponential backoff.
- **Be idempotent.** Ocean.io offers no idempotency key on the request side, and the same payload can
  be delivered more than once. Deduplicate on `personId` in a real store, not an in-memory set.
- There is no webhook signature. The only published authentication is a secret you embed in the
  `webhookUrl` query string and validate yourself, or an IP allowlist you must request from support.

Email `status` values: `verified` (SMTP-confirmed, best for cold outreach), `guessed`
(high-confidence pattern match), `catchAll` (domain accepts everything — avoid in high-volume
campaigns), `notFound` (not charged).

## Errors

| Status | Meaning | Action |
|---|---|---|
| 400 | Token in both header and query param | Use one transport |
| 402 | Out of credits | Nothing was processed and nothing was charged — top up |
| 403 | Token missing, invalid, or not entitled | Regenerate in Account Settings |
| 422 | Bad enum or country-code format | Read `detail[]`, fix `loc`, refetch valid values from `/v2/data-fields` |
| 429 | Rate limited | Honour `Retry-After`; back off exponentially |

Full catalogue: `errors/ocean-io-problem-types.yml`.
