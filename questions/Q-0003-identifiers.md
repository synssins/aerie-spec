# Q-0003: Identifier mapping between aerie and the compatible surface

Raised by: the compatibility lab, first run against a real aerie image.
Status: answered in pull request `phase1-rulings-1`.

## Where

Data Center compatible surface: content, attachment and space ids in paths, bodies and responses, and `userKey` on users. The lab seeds fixtures through `/api/v1`, where every id is a UUID, then calls `/rest/api`.

## Question

How do aerie's own ids appear as content, space and attachment ids on the compatible surface? Cover their format, stability and type, so that a lab fixture seeded through `/api/v1` can be addressed through `/rest/api`.

## Answer

- Content ids, including attachments, are decimal strings from one site-wide sequence, never above 2^53 - 1. Space ids are integers from their own sequence. `userKey` is the user's aerie id as 32 lower-case hex digits.
- Ids are assigned at creation, or at migration for older objects. They never change and are never reused.
- Wherever the compatible surface reads a content id from the client, it also accepts the aerie UUID. Responses always give the decimal id. A fixture is found at `/rest/api/content/{uuid}`, and the response gives its compatible id. Spaces are addressed by key.

Written in `behaviour/common.md` (Identifiers), `behaviour/content.md` and `behaviour/users.md`, with the `contentId`, `attachmentId` and `newContentId` parameters and the `Content` and `Space` id descriptions in `contracts/datacenter/openapi.yaml`.
