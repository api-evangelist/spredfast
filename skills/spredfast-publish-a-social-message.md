---
name: spredfast-publish-a-social-message
description: >-
  Publish a social message through the Spredfast (Khoros Marketing)
  Conversations API — resolve the initiative, resolve the account set, then
  publish to the target accounts and confirm the per-account publication result.
api: Spredfast Conversations API (v2)
base: https://api.spredfast.com/v2
spec: openapi/spredfast-conversations-api-openapi.yml
operations:
  - retrieve-a-list-companies-with-this-app-enabled
  - retrieve-a-list-of-available-initiatives
  - accountset
  - publish-a-new-message-imageset
  - retrieve-an-existing-message
generated: '2026-08-13'
method: generated
source: openapi/spredfast-conversations-api-openapi.yml + https://developer.khoros.com/khorosmarketingdevdocs
---

# Publish a social message

Publishing in Spredfast is never a single call. A message belongs to an
**initiative** (a campaign container) and is targeted at **accounts** that are
made available to that initiative through an **account set**. You must resolve
both before you can publish.

## Before you start

- You need an OAuth 2.0 access token. Get one via the authorization code flow at
  `https://login.spredfast.com/v3/oauth/authorize`, exchanging the code at
  `https://login.spredfast.com/v3/oauth/token`. Tokens are valid for 24 months.
- The DevCenter application must already be enabled for the company and your
  client id allow-listed. That is a support ticket, not a self-serve step.
- Present the token as a bearer token on every call below.

## Steps

1. **Confirm which company you are acting for.**
   `GET /me` — `retrieve-a-list-companies-with-this-app-enabled`.
   Returns the companies this application is enabled for. If this comes back
   empty, the DevCenter application has not been enabled for the company and no
   later step will work.

2. **Resolve the initiative.**
   `GET /conversations/initiative` — `retrieve-a-list-of-available-initiatives`.
   Pick the `id` of the initiative you are publishing into. Use
   `GET /conversations/initiative/{initiativeId}`
   (`retrieve-a-single-initiative`) if you need to confirm one you already hold.

3. **Resolve the target accounts.**
   `GET /conversations/initiative/{initiative-id}/accountset` — `accountset`.
   Returns every account set and, inline, the accounts inside it. Each account
   carries `id`, `service` (FACEBOOK, TWITTER, …), `accountType` (PAGE, USER),
   `alias` and `uniqueId`. Collect the `id` values you want to publish to —
   these become `targetAccountIds`.

   Note the path parameter here is spelled `initiative-id`, not `initiativeId`
   as it is on every other operation. That is a real inconsistency in the
   contract, not a typo in this skill.

4. **Publish.**
   `POST /conversations/initiative/{initiativeId}/message` —
   `publish-a-new-message-imageset`.
   The body is a `Message` object. For a text post:

   ```json
   {
     "sfEntityType": "Message",
     "service": "FACEBOOK",
     "content": { "sfEntityType": "Status", "text": "Status Update" },
     "targetAccountIds": ["4"]
   }
   ```

   `content` takes one of four shapes, discriminated by `sfEntityType`:
   `Status` (text), `ImageShare` (one image), `ImageShareList` (several) or
   `Video`. Set `scheduledPublishDate` to schedule rather than publish now.

   For an upload, use the separate multipart operation
   `POST /conversations/initiative/{initiativeId}/message?multipart`
   (`publish-a-new-message-multipart`) — in this API multipart is a distinct
   path in the contract, not a content-type negotiation on the same one.

   **Video is different.** A video must be uploaded into the Content Center
   first (see the content-center skill) and referenced; it cannot be published
   in one call the way text and images can.

5. **Confirm the outcome per account.**
   `GET /conversations/initiative/{initiativeId}/message/{messageId}` —
   `retrieve-an-existing-message`.
   Read `publicationResults[]`. Publishing to several accounts can partially
   succeed — a 200 on step 4 means Spredfast accepted the message, not that
   every network took it.

## Rules that apply to every step

- **There is no idempotency mechanism.** No `Idempotency-Key`, no replay
  protection. If step 4 times out you do **not** know whether it published.
  Re-issuing it will publish a second post. Recover by listing calendar messages
  (`GET /conversations/calendar/messages`, `get-messages`) and checking for the
  message before retrying. See `conventions/spredfast-conventions.yml`.
- **Errors are not RFC 9457.** A failure comes back as
  `{"status": {"error": {"code": "...", "message": "..."}, "succeeded": false}}`.
  Branch on `status.succeeded`, not on the presence of a `data` key. Some
  failures are not JSON at all — an expired token returns an HTML page reading
  "Developer Inactive", and a bad path returns a non-standard `586 Service Not
  Found`. See `errors/spredfast-problem-types.yml`.
- **No rate limits are published** and no rate-limit headers are returned, so
  there is no budget signal to read. Throttle conservatively on your own side.
  See `rate-limits/spredfast-rate-limits.yml`.
- **Check your privileges first if a call 403s.**
  `GET /conversations/privilege` (`retrieve-permissions-for-current-user`)
  returns the acting user's effective permissions.

## Getting told when it goes live

Publishing is asynchronous with respect to the social network. Rather than
polling step 5, subscribe to the `message-published` event through the
Notifications API — see the event-subscription skill.
