---
name: Segment a customer list and score new accounts against it
description: >-
  Cluster a set of customer domains into named segments, steer the model with positive and negative
  examples, then attribute new domains to the closest segment with a 0-1 fit score.
api: openapi/ocean-io-api-openapi.yml
operations:
  - createSegmentation
  - getSegmentation
  - addMarkedDomains
  - attributeSegmentationDomains
generated: '2026-08-13'
method: generated
source: openapi/ocean-io-api-openapi.yml (operationIds and schema fields verified verbatim)
---

# Segment a customer list and score new accounts against it

Base URL `https://api.ocean.io`. Auth is the `X-Api-Token` header. This is the only stateful
resource in the API — a segmentation is created, processed asynchronously, then queried by id.

## Step 1 — Create the segmentation

`createSegmentation` — `POST /v2/segmentation`

Body is a `SegmentationInput`: `domains` is a representative set of the companies you want grouped —
typically your customers or best-fit accounts — plus optional `leadScoringFeatures` to choose which
signals drive the clustering.

The call runs **asynchronously**. Keep the returned `segmentationId`.

## Step 2 — Poll until it completes

`getSegmentation` — `GET /v2/segmentation/{segmentation_id}`

Poll until `status` is `SUCCESSFUL`. `status` is a `SegmentationStatus` enum with exactly three
values: `IN_PROGRESS`, `SUCCESSFUL`, `FAILED`. Handle `FAILED` explicitly — it is not a transient
state to retry forever.

On success the response carries `segments[]` (each with `segmentId`, `name`, `domains`,
`companyCount`, `traits`, `crmMetrics`, `lookalikeCount`) plus `totalAddressableMarket` and
`totalUntouched`.

There is no webhook for segmentation — this is the one flow in the API you poll rather than
receive. Back off between polls; you still spend rate-limit budget (60 requests/minute self-serve)
on every check.

## Step 3 — Steer the model

`addMarkedDomains` — `POST /v2/segmentation/{segmentation_id}/markDomains`

Append domains to the positive or negative list: `{"domains": [...], "type": "positive"}` or
`"negative"`. Positive domains pull future results toward that shape; negative domains push away
from it. This is how you encode "closed-won looks like this, churned looks like that" without
rebuilding the segmentation.

## Step 4 — Score new accounts

`attributeSegmentationDomains` — `POST /v2/segmentation/{segmentation_id}/attribute-domains`

Body: `{"domains": [...]}`, **maximum 1,000 domains, minimum 1**. The segmentation must have
completed successfully first — calling this early returns `412 Precondition Failed`.

Response: `results[]` with one `AttributedDomain` per input domain **in the same order**, each
carrying `domain`, `segmentId` and a 0-1 `score`, plus `totalRequested` and `totalAttributed`.
Compare those two counts — a domain that could not be attributed is silently absent from the
difference, not an error.

This operation also declares `502 Bad Gateway`, the only operation in the API that does. Treat it as
retryable after a short delay.

## Errors specific to this flow

| Status | Cause |
|---|---|
| 412 | Segmentation has not completed successfully yet |
| 502 | Upstream failure during attribution — retry after a short delay |
| 422 | More than 1,000 domains, or an empty `domains` array |

Full catalogue: `errors/ocean-io-problem-types.yml`.
