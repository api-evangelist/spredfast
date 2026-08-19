---
name: spredfast-export-analytics
description: >-
  Pull analytics out of Spredfast (Khoros Marketing) with the Analytics
  Reporting API — discover the available fields, start an asynchronous export
  job, poll it to completion, and collect the result.
api: Spredfast Analytics Reporting API (beta)
base: https://api.spredfast.com/v2/analytics
spec: openapi/spredfast-analytics-api-openapi.yml
operations:
  - list-all-available-streams-for-a-company
  - get-available-report-fields
  - published-post
  - daily-account-level-metrics
  - daily-post-summary
  - ads-net-values
  - stream-item-data
  - stream-item-action-data
  - custom-feedback
  - profiles
  - retrieve-export-status
  - surveys-list-all-survey-data-for-a-set-of-streams
generated: '2026-08-13'
method: generated
source: openapi/spredfast-analytics-api-openapi.yml + https://developer.khoros.com/khorosmarketingdevdocs/docs/calling-the-analytics-reporting-api
---

# Export analytics

The Analytics Reporting API is **asynchronous by design**. You do not GET a
report — you POST to start an export job, then poll for its status. Every one of
the eight export endpoints follows the same shape, so learn it once.

The docs label this API beta. Its OpenAPI document is titled
`beta-analytics-api`.

## Steps

1. **Find out what you can report on.**
   `GET /streams` — `list-all-available-streams-for-a-company`.
   Returns the streams available to the company. Most exports are scoped by
   stream.

2. **Find out what fields the export supports.**
   `GET /export/{export_type}/fields` — `get-available-report-fields`.
   Do this before building a request body. Field availability differs per export
   type, and there is no static field reference in the docs to work from.

3. **Start the export.**
   POST to the endpoint for the data you want. All of them live under
   `/export/`:

   | Operation | Path | Data |
   |---|---|---|
   | `published-post` | `POST /export/posts` | Published post metrics |
   | `daily-account-level-metrics` | `POST /export/accounts` | Daily per-account metrics |
   | `daily-post-summary` | `POST /export/daily_post_summary` | Daily post rollup |
   | `ads-net-values` | `POST /export/net_ads` | Paid / ads net values |
   | `stream-item-data` | `POST /export/stream_items` | Stream item data |
   | `stream-item-action-data` | `POST /export/stream_item_actions` | Actions taken on stream items |
   | `custom-feedback` | `POST /export/customer_feedback` | Customer feedback |
   | `profiles` | `POST /export/profiles` | Profile data |

   Several of these accept `email_recipients`, `header`, `delimiter` and
   `format` parameters — the export can be delivered as a delimited file rather
   than collected over the API.

   Keep the export `id` from the response.

4. **Poll to completion.**
   `GET /export/{id}/status` — `retrieve-export-status`.
   There is no webhook for export completion, and no `Retry-After` to pace
   against. Poll on your own schedule with backoff.

5. **Surveys are the exception.**
   `GET /surveys` — `surveys-list-all-survey-data-for-a-set-of-streams` — is
   synchronous. It returns survey data for a set of streams directly, with no
   export job.

## Rules

- **No rate limits are published** and no rate-limit headers are returned, so
  your polling loop has no signal to back off against. The export/poll model is
  the only throughput control that exists here — it bounds work per job, not per
  request. Poll with exponential backoff.
- **Errors use the platform envelope**, not RFC 9457:
  `{"status": {"error": {"code", "message"}, "succeeded": false}}`. Branch on
  `status.succeeded`. See `errors/spredfast-problem-types.yml`.
- **No idempotency.** Re-POSTing an export starts a second job. Track the `id`
  you were given rather than retrying blind.
- **Auth is the same OAuth 2.0 bearer token as the rest of the family** —
  `https://login.spredfast.com/v3/oauth/authorize` then `/v3/oauth/token`. A
  single scope, `all`, covers read and write across every endpoint, so there is
  no read-only credential you can issue for a reporting job. See
  `scopes/spredfast-scopes.yml`.
