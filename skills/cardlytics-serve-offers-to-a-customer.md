---
name: Serve Cardlytics offers to a bank customer
description: >-
  Open a customer-scoped Cardlytics Publisher API v2 session and retrieve the ranked,
  targeted card-linked offers to render inside a bank's app or web experience, then read
  that customer's reward summary and redemption history.
api: openapi/cardlytics-publisher-api-openapi.yml
operations:
  - startSession
  - getAds
  - getCustomerProfile
  - getRewardsSummary
  - getCustomerAdRedemptions
generated: '2026-08-12'
method: generated
source: >-
  openapi/cardlytics-publisher-api-openapi.yml (operationIds verified verbatim),
  https://docs.cardlytics.com/ads/v2/getting-started/get-started.html,
  https://docs.cardlytics.com/ads/v2/getting-started/get-session-token.html,
  https://docs.cardlytics.com/ads/v2/error-handling/index.html
---

# Serve Cardlytics offers to a bank customer

You are calling the **Cardlytics Publisher API v2** as a financial institution. This is not a
self-service API: before any of this works, Cardlytics must have signed your client certificate
and allow-listed your source IPs.

## Preconditions

- A Cardlytics-signed mTLS client certificate (`client.crt`) and its private key (`client.key`).
  Pre-production and production use **different** certificates with different subjects.
- Your source IPs allow-listed by Cardlytics.
- A `sourceCustomerId` — the customer identifier **your** institution issues. The customer must
  already have been onboarded through the Data APIs; ads will not serve for an unknown customer.

Base URLs (see `sandbox/cardlytics-sandbox.yml`):

| Environment | Host |
|---|---|
| Sandbox (US) | `https://pub-api-us.sandbox.cardlytics.com` |
| Sandbox (EU/UK) | `https://pub-api-eu.sandbox.cardlytics.com` |
| Production (US) | `https://pub-api-us.prod.cardlytics.com` |
| Production (EU/UK) | `https://pub-api-eu.prod.cardlytics.com` |

## Step 1 — open a customer-scoped session (`startSession`)

`POST /v2/session/startSession`

```json
{ "scopes": ["api:customer"], "sourceCustomerId": "520b3ebc-6496-47f0-863e-d95d667b0cda" }
```

Returns `{ "sessionToken": "..." }` — an encoded JWT carrying the institution and customer
context. Use `api:institution` instead (and omit `sourceCustomerId`) only when you are doing
server-side onboarding through the Data APIs, not offer serving.

**Cache the token.** Cardlytics returns a bare `401` when a session expires with no distinguishing
body, so treat any `401` on a later call as "session expired": call `startSession` again once and
retry. Multiple concurrent sessions per user are acceptable.

## Step 2 — set the session and trace headers on every later call

```
X-CDLX-Session-Token: ${sessionToken}
X-CDLX-Request-Id: <uuid you generate>
Content-Type: application/json
```

`X-CDLX-Request-Id` is echoed back as `requestId` on every response and in every error body. Log
it — it is the only correlation handle Cardlytics support will ask for. If you omit it, Cardlytics
generates one and you lose the ability to match your logs to theirs.

## Step 3 — fetch ranked offers (`getAds`)

`POST /v2/ads/getAds`

The request body (`GetAdsRequest`) accepts channel, geo coordinates, `categoryIds` and
`curationIds`. The response (`GetAdsResponse`) carries `ads[]`, `adRankings`, `categoryGroups` and
`storeLocations`, plus the echoed `requestId`.

Render offers in the order Cardlytics returns them — the ranking is the product.

## Step 4 — enrich the surface

- `GET /v2/customerProfile/getCustomerProfile` (`getCustomerProfile`) — the externally exposed
  customer profile, including accounts, portfolio and opt-in state.
- `GET /v2/customerProfile/getCustomerRewardSummary` (`getRewardsSummary`) — reward totals by
  window (year to date, previous year, previous month, rolling twelve months, current month,
  pending).
- `GET /v2/customerProfile/getCustomerAdRedemptions` (`getCustomerAdRedemptions`) — the customer's
  redemption history joined to the ads that produced it.

## Step 5 — report activity back

Log impressions, activations and interactions through the Events API so the platform can model the
session. This is not optional if you want ranking quality — the events feed targeting.

## Rules that bite

- **No idempotency key exists anywhere on this API.** Do not blind-retry a POST. On a timeout,
  re-issue with the same `X-CDLX-Request-Id` and reconcile from the response, do not assume
  the first call failed. See `conventions/cardlytics-conventions.yml`.
- **Most endpoints accept POST and `application/json` only.** A `405` means you used the wrong
  verb; a `415` means the wrong media type.
- **There is no rate-limit header.** Cardlytics publishes no `X-RateLimit-*`, `RateLimit-*` or
  `Retry-After`. Treat `429` as the only signal and back off exponentially.
- **Error bodies are not RFC 9457.** Expect `{type, title, status, validationErrors[], requestId}`.
  `ValidationException` (400) names offending fields in `validationErrors`;
  `SessionValidationException` means re-authenticate. See `errors/cardlytics-problem-types.yml`.
- **The published OpenAPI declares no 4xx/5xx responses at all** for these five operations. Do not
  generate error handling from the spec — use the error catalog artifact.

## Sandbox behaviour

The sandbox returns pre-defined canned responses; ads created there never actually serve. Any
source customer identifier works for creating a sandbox session. There are no magic test card
numbers. Verify connectivity first:

```sh
curl https://pub-api-eu.sandbox.cardlytics.com/v2/session/startSession \
  --cert client.crt --key client.key \
  -H "Content-Type: application/json" \
  -d '{ "scopes": ["api:institution"] }'
```
