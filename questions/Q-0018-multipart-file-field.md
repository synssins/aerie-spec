# Q-0018: Attachment uploads from generated clients

Raised by: the compatibility lab, run against all 52 T1 operations with clients generated from each pinned published file.
Status: answered in pull request `phase1-rulings-3`.

## Where

Data Center compatible surface: `POST /rest/api/content/{id}/child/attachment/{attachmentId}/data` (`updateAttachmentData`), and the same pattern in `POST /rest/api/content/{id}/child/attachment` (`createAttachments`).

## Question

In every pinned published file, the multipart body of both upload operations declares `file` as a plain string field. Clients generated from those files therefore send `file` as an ordinary form field: a part named `file` with no file name. Aerie answers `400` ("Exactly one file part is needed."). Raw HTTP and our own contract's client pass, because they send a file name. What should the compatible surface accept, without weakening the upload?

## Answer

A part named `file` is the file, whether or not its `Content-Disposition` carries a file name. Its bytes, as received, are the content. All seven pinned files (L-0001 to L-0007) declare `file` the same way, as a string with no binary format, so every client generated from them sends the part without a file name. Those clients have no other way to send the file.

What aerie does with such a part:

- **New bytes for an attachment** (`updateAttachmentData`): the part is the one file part. The attachment's title and media type stay as they are. Any `Content-Type` on the part is ignored, since a generated client may label a plain field as text whatever it holds.
- **New attachments** (`createAttachments`): the request carries no name for the file. Aerie names the attachment `attachment-` followed by its new id, so the name never clashes with an existing one. The media type is `application/octet-stream`. The client can rename it later with the property update or the move operation.
- A part with a file name works as before: the name is the title, and its `Content-Type` gives the media type.

Nothing else is loosened. The body must still be `multipart/form-data` with the XSRF header. The size limit and `413` still apply. Exactly one `file` part is still required for new bytes, and zero or more than one gives `400`. `comment` parts must still match the files one for one. The response does not change.

The lab's finding stands, and aerie's current answer changes: the `400` becomes a success for a `file` part without a file name. The contract keeps `file` as binary, which is what our own client sends. Its descriptions now say that a part without a file name is accepted.

The rest of the sweep: these are the only two multipart bodies among the 52 T1 operations. Their `comment` field is a string in the published files and a list in our contract for `createAttachments`. On the wire a list of one is one part, so a generated client's single comment is accepted as it is.

Written in `behaviour/attachments.md` (Uploading, Uploading new bytes); `createAttachments`, `updateAttachmentData`, `AttachmentUpload` and `AttachmentDataUpload` in the contract.
