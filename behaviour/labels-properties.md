# Labels and properties: Data Center compatible

Sources: L-0001, L-0007, L-0015

T1 covers labels on content. Space labels, label lists (`/rest/api/label/...`), and content and space properties are T2 and will be added to this file then. Shared rules are in `common.md`.

## The label object

- `prefix`: the label's namespace. `global` (shared by everyone, the default), `my` (personal to one user), `team` (shared within a team). Aerie refuses any other prefix on write with `400`.
- `name`: lower-case text without whitespace. Aerie lower-cases names on write and refuses, with `400`, names that contain whitespace or any of `: ; , . ? & [ ] ( ) # ^ * @ ! < >`, or are longer than 255 characters. This character set is aerie's own choice; see open questions.
- `id`: a string of digits, unique for each prefix and name pair on the site.
- `label`: the name as the user sees it, which is `name` for `global` labels and `prefix:name` otherwise.

## Listing a content item's labels

`GET /rest/api/content/{id}/label`

- `prefix` (repeatable): only labels with these prefixes. Empty means all the caller can see. `my` labels are visible only to their owner.
- Offset paging, default `limit` 200.
- Order: by name.

## Adding labels

`POST /rest/api/content/{id}/label`

- Body: either one label object or an array of them. Each has `name` and optionally `prefix` (default `global`). A `name` written as `prefix:name` is split.
- Adding a label already on the content is not an error and adds nothing.
- An invalid name gives `400` and adds none of the labels in the request.
- Result: `200` with the content's complete label list after the change, in the paged shape.
- `403` when the caller may not edit the content.

## Removing a label

`DELETE /rest/api/content/{id}/label?name={label}` and `DELETE /rest/api/content/{id}/label/{label}` do the same thing. The query form exists because a path segment cannot carry `/`; label names with `/` must use it.

- `name` may be given with a prefix (`my:todo`). Without one, `global` is assumed.
- Removing a label that is not on the content gives `404`.
- Result: `204`. `403` when the caller may not edit the content.

## Open questions

- The exact set of characters a real server refuses in label names is not in the sources read so far. The lab should record which names a real server accepts so aerie's rule can match.
