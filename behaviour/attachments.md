# Attachments: Data Center compatible

Sources: L-0001, L-0004, L-0007, L-0015

Covers uploading, listing, updating, moving and removing attachments and their versions, and reading extracted text. Shared rules are in `common.md`; the attachment is a content object, so `content.md` applies unless this file says otherwise.

## The attachment object

An attachment is content with `type: attachment` and these differences:

- `title` is the file name.
- `container` is the page or blog post it belongs to.
- `extensions`: `mediaType` (the MIME type), `fileSize` (bytes, integer), `comment` (the upload note, may be empty).
- `metadata`: `mediaType` and `comment`, the same values as in `extensions`, plus `labels` when expanded.
- `_links.download`: path (relative to `_links.base`) that returns the bytes of the current version. Fetching it with the same credentials returns the file with its stored `Content-Type`. Adding `?version=n` returns version `n`.
- `version`: each upload of new bytes makes a new version, numbered from 1.

## Listing

`GET /rest/api/content/{id}/child/attachment`

- Lists the current attachments of the content. Offset paging, default `limit` 50.
- `filename`: only the attachment with that exact file name (case-sensitive).
- `mediaType`: only attachments of that MIME type.
- Result: `200`. (The published spec lists `201` for this read; aerie returns `200`, see open questions.)

## Uploading

`POST /rest/api/content/{id}/child/attachment`

- `multipart/form-data` with the XSRF header from `common.md`.
- The content may be a page or a folder (see the tree mapping in `content.md`); clients cannot tell them apart, so both accept attachments (question Q-0006).
- One or more parts named `file`. Each one is a file: its bytes, as received, are the attachment's content. The part's `Content-Type` becomes the media type; when absent or `application/octet-stream`, aerie guesses from the file name.
- A `file` part with no file name in its `Content-Disposition` is still a file (question Q-0018). Clients generated from the published files send `file` this way, because those files declare it as a plain string field. Such a part is named `attachment-` followed by the new attachment's id, so it never clashes with an existing name; the client can rename it later. Its `Content-Type`, if any, is ignored, and its media type is `application/octet-stream`.
- Optional parts named `comment`: if any are sent, there must be one per file, in the same order; otherwise `400`.
- Optional parts `minorEdit` (`true`/`false`) and `hidden` (`true`/`false`) apply to all files in the request.
- A file whose name already exists on the content gives `400` unless `allowDuplicated=true`, in which case it becomes a new version of the existing attachment.
- `status`: `current` (default) or `draft`, the status of the container to attach to.
- Result: `200` with a paged list of the created attachments (`results`, `size`, ...), even for one file.
- Errors: `403` when the caller may not add attachments, `404` for unknown content, `413` when a file is over the configured size limit (default 100 MB).

## Updating properties

`PUT /rest/api/content/{id}/child/attachment/{attachmentId}`

- JSON body: an attachment object with `id`, `type: attachment`, `version.number` (current + 1), and the fields to change: `title` (renames the file), `metadata.mediaType`, `metadata.comment`, `container.id` (moves it to other content).
- A wrong `version.number` gives `409`. Renaming to a name already used on the content gives `400`.
- Result: `200` with the attachment. The bytes do not change, but a new version is recorded.

## Uploading new bytes

`POST /rest/api/content/{id}/child/attachment/{attachmentId}/data`

- Same multipart rules as upload, with exactly one `file` part, and optional `comment` and `minorEdit` parts. A `file` part without a file name counts (question Q-0018). Zero `file` parts, or more than one, gives `400`.
- Makes a new version of the attachment. The file name in the part becomes the new title. When the part has no file name, the title and media type stay as they are, and any `Content-Type` on the part is ignored.
- Result: `200` with the attachment.

## Moving

`POST /rest/api/content/{id}/child/attachment/{attachmentId}/move`

- Query parameters: `newContentId` (content to move to; omitted means stay), `newName` (new file name; omitted means keep). At least one is required, otherwise `400`.
- Needs the XSRF header.
- The target must be a current page (an aerie page or folder) the caller can add attachments to. A name clash at the target gives `400`.
- Result: `204`.

## Removing

`DELETE /rest/api/content/{id}/child/attachment/{attachmentId}` moves the attachment to the space trash (status `trashed`), with all its versions. Result: `204`.

`DELETE /rest/api/content/{id}/child/attachment/{attachmentId}/version/{version}` removes one version for good. Removing the only version gives `400`; remove the attachment instead. Result: `204`.

Both give `404` when the attachment is not on the content named in the path.

## Extracted text

`GET /rest/api/content/{id}/child/attachment/{attachmentId}/extractedtext` (since 10.0.1)

- Returns `{ "extractedText": "..." }`: the plain text aerie's indexer pulled from the current version.
- `limit` (integer): the most characters to return; the text is cut at that length.
- `204` with no body when there is no extracted text yet or the file type has none.

## Open questions

- The published spec lists `201` as the success status of the attachment list read. Aerie returns `200` because it is a read; the lab will record what real servers send.
- Whether upload returns a paged list or a bare array on real servers. Aerie returns the paged list shape.
