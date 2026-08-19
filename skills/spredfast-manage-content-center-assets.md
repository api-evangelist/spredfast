---
name: spredfast-manage-content-center-assets
description: >-
  Organise and upload assets in the Spredfast (Khoros Marketing) Content
  Center — create folders, upload text, image and video assets, list what is in
  a folder, and prepare a video for publishing.
api: Spredfast Conversations API (v2)
base: https://api.spredfast.com/v2
spec: openapi/spredfast-conversations-api-openapi.yml
operations:
  - create-a-new-content-asset-folder
  - retrieve-a-list-of-folders
  - retrieve-a-content-asset-folder-by-id
  - update-existing-content-center-asset-folder
  - delete-a-content-asset-folder-by-id
  - retrieve-a-list-of-assets-in-a-folder
  - assets-create-text-asset
  - assets-create-image-or-video
  - assets-retrieve-a-content-center-asset-by-id
  - assets-update-existing-content-center-asset
  - create-a-video-captions
generated: '2026-08-13'
method: generated
source: openapi/spredfast-conversations-api-openapi.yml + https://developer.khoros.com/khorosmarketingdevdocs
---

# Manage Content Center assets

The Content Center is where reusable creative lives. It matters most for
**video**: unlike text and images, a video cannot be published in a single call
— it has to be uploaded here first.

## Folders

- `POST /conversations/folder` — `create-a-new-content-asset-folder`
- `GET /conversations/folder` — `retrieve-a-list-of-folders`
- `GET /conversations/folder/{folderId}` — `retrieve-a-content-asset-folder-by-id`
- `PUT /conversations/folder/{folderId}` — `update-existing-content-center-asset-folder`
- `DELETE /conversations/folder/{folderId}` — `delete-a-content-asset-folder-by-id`
- `GET /conversations/folder/{folderId}/assets` — `retrieve-a-list-of-assets-in-a-folder`

## Assets

**Text asset** — `POST /conversations/asset`, `assets-create-text-asset`.
A normal JSON body.

**Image or video** — `POST /conversations/asset?multipart`,
`assets-create-image-or-video`.
This is a *separate operation at a separate path*, not a content-type variant of
the one above. The `?multipart` query flag is part of the path in the contract.

**Read one back** — `GET /conversations/asset/{assetId}`,
`assets-retrieve-a-content-center-asset-by-id`.

**Update** — `PUT /conversations/asset/{assetId}`,
`assets-update-existing-content-center-asset`.

Assets carry `createdByUserId` and `lastModifiedByUserId`, so you can attribute
them, and `folderId`, which is how they belong to a folder.

## Publishing a video

1. Upload the video with `assets-create-image-or-video`
   (`POST /conversations/asset?multipart`). Keep the returned asset id.
2. Optionally add captions:
   `POST /conversations/initiative/{initiativeId}/video/captions` —
   `create-a-video-captions`.
3. Publish, referencing the asset, with the publishing skill
   (`spredfast-publish-a-social-message`). The message `content` takes
   `"sfEntityType": "Video"`.

Note that the captions operation is declared under an initiative but the
harvested contract carries a validation warning from the docs platform: the
path is missing its `{initiativeId}` parameter declaration. Supply the
initiative id in the path anyway — the surrounding operations all require it.

## Rules

- **No idempotency.** A re-tried multipart upload creates a second asset. On a
  timeout, list the folder (`retrieve-a-list-of-assets-in-a-folder`) before
  re-uploading.
- **No published upload size limit, no rate limit, and no rate-limit headers.**
  See `rate-limits/spredfast-rate-limits.yml`.
- **Errors use the platform envelope**, not RFC 9457, and a malformed body
  returns `406` with code `not_acceptable` and message "Unable to parse the json
  you provided." See `errors/spredfast-problem-types.yml`.
