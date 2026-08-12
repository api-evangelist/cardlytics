---
name: Ingest merchants and offers into Cardlytics
description: >-
  Authenticate as a Cardlytics advertiser partner with OAuth 2.0 client credentials, then
  upsert and delete merchant and offer records through the Partner API using your own
  external identifiers.
api: openapi/cardlytics-partner-api-openapi.yml
operations:
  - POST /api/v1/idp/oauth2/token
  - PUT /api/v1/partner/merchants/{external_merchant_id}
  - DELETE /api/v1/partner/merchants/{external_merchant_id}
  - PUT /api/v1/partner/merchants/{external_merchant_id}/offers/{external_offer_id}
  - DELETE /api/v1/partner/merchants/{external_merchant_id}/offers/{external_offer_id}
generated: '2026-08-12'
method: generated
source: >-
  openapi/cardlytics-partner-api-openapi.yml (paths and payload shapes verified verbatim),
  https://platform.cardlytics.com/advertisers/docs/api-merchant-rest-api,
  https://platform.cardlytics.com/advertisers/docs/api-get-started,
  https://platform.cardlytics.com/advertisers/docs/api-faqs
note: >-
  The Cardlytics Partner API OpenAPI declares NO operationId on any operation, so this skill
  binds steps to METHOD + path, which is what the spec actually publishes. Do not invent
  operationIds for it.
---

# Ingest merchants and offers into Cardlytics

You are an **advertiser partner** pushing merchant and offer data into the Cardlytics commerce
media platform. Credentials are issued by your Cardlytics representative — there is no sign-up.

| Environment | Host |
|---|---|
| Sandbox | `https://api-sandbox.cardlytics.com` |
| Production | `https://api.cardlytics.com` |

The spec labels production "Documentation reference only, contact support for access". Both hosts
return `403 {"message":"Forbidden"}` to anonymous callers — access is IP-allow-listed as well as
OAuth-gated.

## Step 1 — get an access token

`POST /api/v1/idp/oauth2/token` with `Content-Type: application/x-www-form-urlencoded`:

```
grant_type=client_credentials&client_id=<your id>&client_secret=<your secret>
```

Returns `{ "access_token": "...", "token_type": "Bearer", "expires_in": <seconds> }`. Send it as
`Authorization: Bearer <access_token>` on every subsequent call. A `401` means the credentials or
the token are invalid — re-issue, do not retry the original call.

> The spec declares the oauth2 `tokenUrl` as the *relative* `/v1/idp/oauth2/token` while the
> actual path is `/api/v1/idp/oauth2/token`. Use the path form above.

## Step 2 — upsert a merchant

`PUT /api/v1/partner/merchants/{external_merchant_id}`

The path segment is **your** merchant identifier, and it must equal `merchantId` in the
`MerchantPayload` body — a mismatch is a `400`. Payload carries `merchantName`,
`merchantCategoryCode`, `merchantUrl`, `merchantSubCategories[]`, `paymentChannels[]` and
`stores[]`, where each store carries address, geo coordinates and the processor MID records per
payment network (Visa `vmid`/`vsid`, Mastercard auth/clearing MIDs + ICA, Amex SE number,
Discover). Those MIDs are how Cardlytics matches real card transactions back to your merchant —
supply as many as you have, including optional fields.

Success is `202 Accepted` returning `{ "message": ..., "trace_id": ... }`. The record is queued,
not applied synchronously. Keep the `trace_id`.

Merchant *status* is not a field: it is inferred from the verb. `PUT` means UPSERT, `DELETE` means
DELETED.

## Step 3 — upsert an offer under that merchant

`PUT /api/v1/partner/merchants/{external_merchant_id}/offers/{external_offer_id}`

Body is an `OfferPayload` (which references `ImageAsset` → `LargeImage` for creative). Same rules:
your own identifiers, path must match body, `202` on acceptance.

## Step 4 — remove records

`DELETE /api/v1/partner/merchants/{external_merchant_id}` and
`DELETE /api/v1/partner/merchants/{external_merchant_id}/offers/{external_offer_id}`, with no
request payload.

**A deleted offer stays live for customers who already activated it**, through the end of its
30–45 day redemption period. Deleting is not a kill switch — plan reconciliation accordingly.

## Rules that bite

- **Send changes only.** Do not re-push unchanged merchants or offers. Cardlytics keeps using the
  last upload; a full daily resync will burn your quota for nothing.
- **One record per request.** There is no batch endpoint.
- **Quotas are per-partner and unpublished.** You are assigned a daily call quota (roughly the
  number of merchant offers you carry) and an RPS ceiling. Exceeding either returns `429`. There
  are **no** rate-limit response headers, so pace yourself against your assigned numbers, not
  against a header. See `rate-limits/cardlytics-rate-limits.yml`.
- **PUT is idempotent by natural key, and that is the only retry safety on this API.** A repeated
  `PUT` with the same body converges on the same state. There is no `Idempotency-Key` header for
  anything else.
- Errors are not RFC 9457. `400` covers both schema validation failure and path/body ID mismatch —
  check both before assuming a schema problem.

## Related

- Reporting and redemption reconciliation: `skills/cardlytics-report-and-reconcile-redemptions.md`
- Conventions: `conventions/cardlytics-conventions.yml`
- Errors: `errors/cardlytics-problem-types.yml`
