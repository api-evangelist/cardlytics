---
name: Pull Cardlytics performance reports and reconcile daily redemptions
description: >-
  Retrieve aggregate merchant/offer performance metrics from the Cardlytics Partner API and
  download the daily redemption feed through its short-lived pre-signed URL, then reconcile
  redemptions against your own offer records.
api: openapi/cardlytics-partner-api-openapi.yml
operations:
  - POST /api/v1/idp/oauth2/token
  - POST /api/v1/partner/merchants/{external_merchant_id}/reports
  - GET /api/v1/partner/redemptions
generated: '2026-08-12'
method: generated
source: >-
  openapi/cardlytics-partner-api-openapi.yml (paths, parameters and response shapes verified
  verbatim), https://platform.cardlytics.com/advertisers/docs/api-faqs
note: >-
  The Cardlytics Partner API OpenAPI declares no operationId on any operation; steps below bind
  to METHOD + path, which is what the spec publishes.
---

# Pull Cardlytics performance reports and reconcile daily redemptions

Authenticate first — `POST /api/v1/idp/oauth2/token` with `grant_type=client_credentials` — and
send `Authorization: Bearer <access_token>` on both calls below. Hosts:
`https://api-sandbox.cardlytics.com` (sandbox), `https://api.cardlytics.com` (production).

## Aggregate performance report

`POST /api/v1/partner/merchants/{external_merchant_id}/reports`

```json
{
  "cube": "merchant_performance",
  "offerIds": [],
  "timeRange": { "from": "2023-10-01", "to": "2025-09-17" }
}
```

- `cube` currently supports **only** `merchant_performance`.
- `offerIds` empty or omitted returns metrics for every offer under the merchant. Cap is
  **1000 offer IDs** per request.
- `timeRange.from` defaults to two years back — that is the maximum lookback, not just a default.
  `timeRange.to` defaults to today (`YYYY-MM-DD`).

The response is a columnar `ReportResponse`: `header.fields[]` names each column and tags it `DIM`
(dimension) or `FACT` (metric), `header.maxRows` (`-1` for unbounded), and `rows[]` as arrays of
raw values positionally aligned to `fields`. **Do not hardcode column positions** — read
`header.fields` and index by `fieldName`. Published fields are Partner Merchant Id, Partner Offer
Id, Impressions, Purchases, Revenue, Reach and Activations.

A `400` here means invalid parameters or an out-of-range date window; a `401` means the token is
missing or invalid.

## Daily redemption feed

`GET /api/v1/partner/redemptions?date=YYYY-MM-DD`

Returns `{ "url": "<pre-signed URL>" }`.

- The `date` must be **in the past in UTC**. Today or a future date is a `400` — this is the most
  common failure, and it bites anyone running the job from a non-UTC timezone shortly after
  midnight local.
- The pre-signed URL is valid for **60 minutes only**. Download immediately; do not persist the
  URL in a queue or retry it hours later.
- One file per day. Contents include an anonymized transaction id, your merchant and offer ids,
  store id, merchant name, card type, last four of the card, transaction amount, and an ISO 8601
  transaction timestamp.

## Reconciliation guidance

- Join redemption rows back to your catalog on **your own** `external_merchant_id` /
  `external_offer_id` — those are the only identifiers shared across ingestion, reporting and
  redemptions.
- An offer you deleted still redeems for customers who activated it before deletion, for the rest
  of its 30–45 day redemption period. Expect redemption rows for offers that are no longer in your
  active catalog and do not treat them as errors.
- Report metrics are aggregate; the redemption feed is per-transaction. They will not tie out
  exactly on the same day because report facts and settled redemptions land on different clocks.

## Rules that bite

- Reporting is a `POST`, and it is **not idempotent-keyed** — but it is a read, so a retry is safe.
  The redemption `GET` is safe to retry within the URL's 60-minute life.
- No rate-limit headers. Your daily quota and RPS are assigned per partner; `429` is the only
  signal. Back off exponentially.
- Both calls return the platform's bespoke error shapes, not RFC 9457. See
  `errors/cardlytics-problem-types.yml`.
