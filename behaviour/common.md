# Common behaviour: Data Center compatible surface

Sources: L-0001, L-0002, L-0003, L-0004, L-0005, L-0006, L-0007, L-0013, L-0014, L-0015

This file holds the rules every Data Center compatible operation shares: base path, authentication, paging, expansion, links, errors, the XSRF header, and the two small system resources in T1 (server information and long tasks). The area files (`content.md`, `spaces.md` and the rest) assume these rules and only state where an operation differs.

## Base path

- The compatible surface lives under a context path chosen at install time, followed by `/rest/api`. A client configured with `https://wiki.example.org/kb` calls `https://wiki.example.org/kb/rest/api/content`.
- Aerie serves the surface at the root context by default (`/rest/api/...`) and can be configured to serve it under a prefix. The prefix appears in `_links.base` and `_links.context` (see below).
- Requests and responses are JSON (`application/json`, UTF-8) unless an operation says otherwise. Attachment upload uses `multipart/form-data`.

## Authentication

- HTTP Basic (`Authorization: Basic base64(username:password)`) is accepted on every operation.
- Other schemes (personal access tokens, OAuth 2.0) are specified in `identity.md` and are out of scope for T1 here.
- A request with no credentials is treated as the anonymous user. If anonymous access is not allowed for what was asked, the response is `401`.
- Bad credentials give `401` regardless of what was requested.

## Hiding what the caller cannot see

- When a read targets something that exists but the caller may not view, the response is `404`, the same as for something that does not exist. A client must not be able to tell the two apart.
- When a write targets something the caller can view but may not change, the response is `403`.
- Lists silently leave out items the caller may not view. `size` counts only what is returned.

## Paging (offset style)

Most list operations take `start` and `limit` query parameters.

- `start` is the zero-based index of the first item to return. Default `0`.
- `limit` is the most items to return. Each operation states its default (usually 25). Aerie also applies a server-wide ceiling (default 200, configurable); a larger `limit` is reduced to the ceiling without error. The reduced value is what the response reports in `limit`.
- A negative `start` or a `limit` below 1 gives `400`.

A paged response is an object with:

| Field | Type | Meaning |
|---|---|---|
| `results` | array | The items on this page |
| `start` | integer | The `start` that was applied |
| `limit` | integer | The `limit` that was applied, after the ceiling |
| `size` | integer | How many items are in `results` |
| `_links` | object | Links (below), including `next` and `prev` where they apply |

- `_links.next` is present only when another page exists. It is the same request path and query with `start` set to `start + limit`.
- `_links.prev` is present only when `start > 0`. It points at `start - limit`, floored at 0.
- Both are paths that start with `/rest/api/`, without the context prefix. A client joins them with `_links.base`.

## Paging (cursor style)

`GET /rest/api/content/scan` and `GET /rest/api/space/{spaceKey}/trash` page with an opaque `cursor` instead of `start`.

- The first request sends no `cursor`. Each response carries `cursor` (the one applied), and `nextCursor` / `prevCursor` where another page exists in that direction, plus `limit`, `size`, `results` and `_links` (`next` / `prev` built with the cursor).
- Cursors are opaque strings to the client. Aerie encodes the sort key of the last item; a cursor that cannot be decoded gives `400`.
- Changes made between requests may or may not show up in later pages; a cursor never makes an item appear twice.

## Expansion

Many fields are left out of a response unless the caller asks for them with `expand`.

- `expand` is a comma-separated list of field paths. Dots reach into nested objects: `expand=body.storage,version,history.lastUpdated`.
- A field that was not expanded is absent from the object. Its name appears as a key in the object's `_expandable` map. The value is either a REST path where the full value can be fetched (for example the space of a page points at `/rest/api/space/KEY`) or an empty string.
- Asking to expand a name that does not exist on that object is ignored, not an error.
- Defaults: a single-item read (for example `GET /rest/api/content/{id}`) expands `space`, `history` and `version` when `expand` is not given. List operations expand nothing by default. Sending `expand` replaces the default; it does not add to it.
- Expansion depth is limited to what is listed for each object in the contract. Aerie does not support unlimited recursion (for example `ancestors.ancestors`); unknown deeper paths are ignored.

## Links

Every top-level object and most nested objects carry `_links`:

| Key | Present on | Meaning |
|---|---|---|
| `self` | every object | Absolute URL of this object's REST resource |
| `base` | top-level response only | Absolute base URL of the site, including any context prefix |
| `context` | top-level response only | The context prefix alone, or empty string |
| `webui` | content, spaces | Path (relative to `base`) of the object's page in the web UI |
| `tinyui` | content | Short path to the object's page in the web UI |
| `edit` | editable content | Path of the editor for the object |
| `download` | attachments | Path (relative to `base`) that returns the file bytes |
| `collection` | single content | REST path of the collection the object belongs to |
| `next`, `prev` | paged responses | See paging |

Paths in `webui`, `tinyui`, `edit` and `download` are aerie's own UI routes. Clients follow them by joining with `base`; they must not rely on their shape.

## Errors

Errors use one body shape (`RestError` in the contract):

| Field | Type | Meaning |
|---|---|---|
| `statusCode` | integer | Same as the HTTP status |
| `message` | string | A human-readable explanation, in aerie's words |
| `reason` | string | The HTTP reason phrase, for example `Not Found` |
| `data` | object | Optional. `valid` (boolean), `allowedInReadOnlyMode` (boolean), `successful` (boolean), `authorized` (boolean), `errors` (array of `{message: {key, args, translation}}`) |

Status codes used across the surface: `200`, `201` (not used by T1), `202` (long-running task accepted), `204`, `400` (bad parameter or body), `401`, `403`, `404`, `405` (write refused in read-only mode, aerie's choice), `409` (version or state conflict), `413` (upload too large), `415` (wrong content type), `500`.

## XSRF header

- Any `POST`, `PUT` or `DELETE` that carries `multipart/form-data` or `application/x-www-form-urlencoded` must carry the header `X-Atlassian-Token` with value `no-check` or `nocheck`. Both spellings appear in published material and both are accepted. Without it the response is `403` and nothing changes.
- JSON requests do not need the header. Sending it anyway is harmless.

## Read-only mode

When an administrator puts aerie into read-only mode, every write returns `405` with a `RestError` whose `data.allowedInReadOnlyMode` is `false`. Reads are unaffected. (Read-only mode itself is a T3 administration surface.)

## Server information

`GET /rest/api/server-information` returns facts about the running build:

- `baseUrl` (string): the configured site URL.
- `version` (string): the Data Center version aerie presents itself as. Aerie reports the highest Data Center version whose T1 surface it fully implements (configurable, default `9.2.4` until the T1 suite passes against later versions). Clients often branch on this value, so it must be a real Data Center version string, not aerie's own version.
- `buildNumber` (integer) and `marketplaceBuildNumber` (integer): numbers consistent with that version. Aerie keeps a table mapping each presented version to its build number.
- `buildDate` (string): an ISO 8601 date-time.

The operation needs an authenticated caller; anonymous callers get `401`.

## Long tasks

Some operations (space delete is the one in T1) start work that outlives the request. They return `202` with a `LongTaskSubmission`:

- `id`: the task id.
- `links.status`: REST path of the task's status resource, `/rest/api/longtask/{id}`.

`GET /rest/api/longtask` lists the tasks the caller started (administrators see all), newest first, offset-paged with default `limit` 100. `GET /rest/api/longtask/{id}` returns one `LongTaskStatus`:

- `id`, `name` (a message object: `key`, `args`, `translation`), `elapsedTime` (milliseconds), `percentageComplete` (0 to 100), `successful` (boolean), `messages` (array of message objects), `_links`.
- While the task runs, `successful` is `false` and `percentageComplete` is below 100. When it ends, `percentageComplete` is 100 and `successful` tells the outcome.
- Aerie keeps finished task records for at least 24 hours. After that `GET /rest/api/longtask/{id}` may give `404`.
- A caller who did not start the task and is not an administrator gets `404`.
