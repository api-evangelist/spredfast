---
name: spredfast-subscribe-to-events
description: >-
  Subscribe to Spredfast (Khoros Marketing) platform events — create a
  subscription for one event name, receive it either pushed to your
  notificationUri or by polling, and manage the subscription lifecycle.
api: Spredfast Notifications (Events) API
base: https://api.spredfast.com/v2/events
spec: openapi/spredfast-notification-api-openapi.yml
operations:
  - create-a-subscription
  - list-subscriptions
  - retrieve-subscription-details
  - retrieve-subscription-events
  - update-a-subscription
  - delete-a-subscription
generated: '2026-08-13'
method: generated
source: openapi/spredfast-notification-api-openapi.yml + https://developer.khoros.com/khorosmarketingdevdocs/docs/notification-tutorials
---

# Subscribe to platform events

The Notifications (Events) API is Spredfast's event surface. It is a plain REST
API for managing subscriptions — there is no AsyncAPI document and no event
catalog endpoint, so the event names below come from the documentation and are
the complete published set.

**One subscription listens for exactly one event name.** To hear about three
things, create three subscriptions.

## Choose a delivery mode

You get to pick, and the choice is made by one field on the subscription:

- **Push** — set `notificationUri`. Khoros POSTs each event to that URL.
- **Pull** — omit `notificationUri` and poll
  `GET /data/{subscriptionId}` instead.

## Steps

1. **Create the subscription.**
   `POST /subscription` — `create-a-subscription`.

   ```json
   {
     "eventName": "message-published",
     "externalId": "your-own-id-for-this-subscription",
     "notificationUri": "https://your-host.example.com/hooks/spredfast",
     "bearerToken": "a-token-you-generate",
     "query": "optional regex filter"
   }
   ```

   - `eventName` — required. One of the published names listed below.
   - `externalId` — your own identifier, so you can reconcile subscriptions
     against your own records.
   - `notificationUri` — optional; set it for push delivery.
   - `bearerToken` — optional. Khoros sends it as a `Bearer` token on the
     outbound call, so **this is how you authenticate the sender**. There is no
     payload signature and no HMAC — if you do not set `bearerToken`, your
     endpoint has no way to verify an inbound event actually came from Khoros.
     Set it.
   - `query` — optional regex, applied server-side to filter events.

   `uuid`, `companyId`, `userId`, `createdDate` and `status` are read-only and
   come back on the response. Keep the `uuid` — it is the subscription id for
   every later call.

2. **Verify it registered.**
   `GET /subscription/{id}` — `retrieve-subscription-details`, or
   `GET /subscription` — `list-subscriptions` (paginate with `pageNumber` and
   `pageSize`).

3. **Receive events.**

   *Push:* handle the POST at your `notificationUri`. Check the `Authorization`
   header against the `bearerToken` you supplied, then read the envelope:
   `companyId`, `eventName`, `createdDate`, `eventUuid`, `type`,
   `subscriptionUuid` and `data`.

   *Pull:* `GET /data/{subscriptionId}` — `retrieve-subscription-events`, with
   `offset` and `limit`. Note both default to `0` in the contract, so pass real
   values.

4. **Pause or resume.**
   `PUT /subscription/{id}/{status}` — `update-a-subscription`. Status is one of
   `ACTIVE`, `DISABLED`, `PAUSED`. Pause rather than delete when you are
   redeploying a receiver.

5. **Delete when finished.**
   `DELETE /subscription/{id}` — `delete-a-subscription`.

## The published events

| eventName | what it means |
|---|---|
| `message-published` | A message went live on the external network. |
| `message-sent-to-network` | Data was saved to the network for future publishing. |
| `message-needs-approval` | A message's state changed to require approval. |
| `rule-applied` | An automation rule fired. |
| `stream-item-label-applied` | A label was applied to a stream item in the inbox. |
| `credential-auth` | A social account credential was authorized. |
| `credential-reauth` | A social account credential was re-authorized. |
| `credential-deauth` | A social account credential was de-authorized. |
| `credential-delete` | A social account credential was deleted. |
| `pass-conversation-control` | Conversation control passed between a bot and an agent. |
| `error-custom-crm` | An outbound call to a customer Custom CRM failed. |

The four `credential-*` events are the ones to wire first if you operate this
platform: they are how you find out a connected social account has stopped
working, before publishing starts failing.

## Rules

- **No retry policy, no delivery guarantee and no signature are documented.**
  Treat delivery as at-most-once unless you have confirmed otherwise with
  Khoros, make your handler idempotent on `eventUuid`, and reconcile against the
  REST API rather than trusting the stream alone.
- **`error-custom-crm` is your only tracing channel.** The API returns no
  request-id or correlation header on any response. The only `trackingId` in the
  whole surface is inside this event's payload, which also carries the full
  failed external call — method, URI, headers, query, body, response headers and
  response code. If you run a Custom CRM integration, subscribe to this event;
  without it, a failed callback is invisible.
- **No event `version` values are published**, although the subscription model
  has a `version` field. Omit it unless Khoros support gives you one.
- Errors follow the platform envelope:
  `{"status": {"error": {...}, "succeeded": false}}`. See
  `errors/spredfast-problem-types.yml`.
