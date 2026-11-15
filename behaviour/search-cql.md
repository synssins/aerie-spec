# Search and the supported CQL subset: Data Center compatible

Sources: L-0001, L-0007, L-0013, L-0014, L-0016, L-0017, L-0018, L-0019

Covers the two search operations and the part of the query language aerie supports in T1. Shared rules are in `common.md`.

## Operations

`GET /rest/api/content/search`

- `cql` (required): the query. Missing or empty gives `400`.
- `cqlcontext`: optional JSON object, URL-encoded, giving the context functions need: `spaceKey` (used by `currentSpace()`) and `contentId` (the content the search is made from). Bad JSON gives `400`; unknown keys are ignored.
- Returns content objects in the usual offset-paged shape (`results`, `start`, `limit`, `size`, `_links`), default `limit` 25. `expand` works as on `GET /rest/api/content`.
- Only content matches; a query that can only match spaces gives an empty list.

`GET /rest/api/search`

- Same `cql`, `cqlcontext`, `start` (default 0), `limit` (default 25) and `expand`.
- `excerpt`: how to build each result's excerpt. `highlight` (default) marks matched terms with `@@@hl@@@` and `@@@endhl@@@` around them; `indexed` gives the start of the indexed text with no marks; `none` leaves `excerpt` empty.
- `includeArchivedSpaces`: default `false`. When false, content in archived spaces is left out.
- Returns a search response:
  - `results`: search result items (below).
  - `start`, `limit`, `size`: as in offset paging.
  - `totalSize`: how many items match in all, before paging. Aerie may cap this count at 1000 and report the cap.
  - `cqlQuery`: the query as aerie ran it.
  - `searchDuration`: milliseconds spent.
  - `archivedResultCount` (optional): how many matches were left out because they are in archived spaces.
  - `_links`: `self`, `base`, `context`, and `next` / `prev` as in offset paging.
- Each result item:
  - `content`: the content object (only `id`, `type`, `status`, `title`, `_links` unless `expand` asks for more; use `expand=content.space` and so on).
  - `title`: the title, with highlight marks when `excerpt=highlight`.
  - `excerpt`: a short passage of the body around the match.
  - `url`: path of the item's page in the web UI, relative to `_links.base`.
  - `resultGlobalContainer`: `{title, displayUrl}` of the space.
  - `resultParentContainer`: `{title, displayUrl}` of the parent page, when there is one.
  - `entityType`: `content` for every T1 result.
  - `iconCssClass`: a CSS class naming the item kind. Clients must treat it as opaque.
  - `lastModified`: ISO 8601 date-time. `friendlyLastModified`: the same date in short human words (for example `Mar 03, 2026`).

Both operations give `400` with a `RestError` naming the problem (unknown field, wrong operator for the field, bad date, unbalanced brackets) when the query cannot be parsed. Matches the caller cannot view are dropped before paging, so they never count in `size` or `totalSize`.

## Query structure

A query is one or more clauses joined by `AND` and `OR`, with optional `NOT` and brackets, and an optional `ORDER BY` at the end.

- A clause is `field operator value`.
- Keywords and operators are case-insensitive. Field names are case-insensitive.
- Without brackets, `AND` and `OR` are evaluated left to right with no precedence between them. Brackets group.
- `NOT clause` is allowed anywhere but at the very start of a query; a query whose first clause is negative gives `400`.
- Values that hold spaces, reserved characters or reserved words must be quoted with `"` or `'`. Inside a quoted value, a quote or backslash is escaped with a backslash.
- Reserved words (must be quoted when used as values): after, and, as, avg, before, begin, by, commit, contains, count, distinct, else, empty, end, explain, from, having, if, in, inner, insert, into, is, isnull, left, like, limit, max, min, not, null, or, order, outer, right, select, sum, then, was, where, update.
- A list value, for `IN` and `NOT IN`, is a bracketed comma-separated list: `space IN ("DEV", "OPS")`.

## Operators

| Operator | Meaning | Field kinds |
|---|---|---|
| `=` | equals | all except text |
| `!=` | does not equal | all except text |
| `>` `>=` `<` `<=` | range | dates |
| `IN` | equals any in the list | all except text and dates |
| `NOT IN` | equals none in the list | all except text and dates |
| `~` | contains (text match) | text |
| `!~` | does not contain | text |

Using an operator a field does not support gives `400`.

## Fields in T1

| Field | Kind | Operators | Notes |
|---|---|---|---|
| `id`, `content` | content id | `=` `!=` `IN` `NOT IN` | `content` is another name for `id` |
| `type` | type | `=` `!=` `IN` `NOT IN` | `page`, `blogpost`, `comment`, `attachment` |
| `space`, `space.key` | space key | `=` `!=` `IN` `NOT IN` | Matched without regard to case |
| `space.type` | space type | `=` `!=` `IN` `NOT IN` | `global`, `personal`, and `favourite` / `favorite` (the caller's favourite spaces) |
| `space.title` | text | `~` `!~` | The space name |
| `title` | text | `=` `!=` `~` `!~` `IN` `NOT IN` | `=` is an exact, case-insensitive match |
| `text` | text | `~` `!~` | Title, body, labels and attachment extracted text |
| `label` | string | `=` `!=` `IN` `NOT IN` | Label name; `my:name` matches the caller's personal label |
| `creator` | user | `=` `!=` `IN` `NOT IN` | Username or user key |
| `contributor` | user | `=` `!=` `IN` `NOT IN` | Anyone who made a version |
| `created` | date | `=` `!=` `>` `>=` `<` `<=` | |
| `lastmodified` | date | `=` `!=` `>` `>=` `<` `<=` | |
| `ancestor` | content id | `=` `!=` `IN` `NOT IN` | Any page above, at any depth |
| `parent` | content id | `=` `!=` `IN` `NOT IN` | The direct parent page |
| `container` | content id | `=` `!=` `IN` `NOT IN` | The page or blog post a comment or attachment belongs to |

A field not in this table gives `400` with a message saying it is not supported. Fields deferred to T2: `macro`, `mention`, `watcher`, `favourite` / `favorite`, `space.category`. Custom fields from add-ons are not supported.

## Values

- Date values: `yyyy-MM-dd`, `yyyy/MM/dd`, `yyyy-MM-dd HH:mm` or `yyyy/MM/dd HH:mm`, quoted or not, read in the site's time zone. A date with no time means the start of that day. `created = 2026-03-03` matches the whole day.
- User values: a username or user key.
- Content id values: decimal digits.

## Functions in T1

| Function | Use with | Result |
|---|---|---|
| `currentUser()` | `creator`, `contributor` | The caller. Anonymous callers match nothing |
| `currentSpace()` | `space` | The `spaceKey` from `cqlcontext`; `400` when absent |
| `now(inc)` | dates | Now, moved by `inc` |
| `startOfDay(inc)`, `endOfDay(inc)` | dates | Start or end of today, moved by `inc` |
| `startOfWeek(inc)`, `endOfWeek(inc)` | dates | Start or end of this week, moved by `inc` |
| `startOfMonth(inc)`, `endOfMonth(inc)` | dates | Start or end of this month, moved by `inc` |
| `startOfYear(inc)`, `endOfYear(inc)` | dates | Start or end of this year, moved by `inc` |

- `inc` is optional, quoted, and has the form sign, number, unit: `"-4w"`, `"+1d"`, `"2M"`. The sign defaults to `+`. Units: `y` year, `M` month, `w` week, `d` day, `h` hour, `m` minute.
- Weeks start on the site's configured first day of the week (Monday by default in aerie).

Deferred to T2: `favouriteSpaces()`, `recentlyViewedContent(limit, offset)`, `recentlyViewedSpaces(limit)`.

## Text matching

`~` and `!~` on `title`, `text` and `space.title`:

- Matching is case-insensitive and works on words, after the same stemming aerie's own search uses.
- Several words without quotes match items holding all of them, in any order.
- A phrase in double quotes inside the value matches the words in that order: `text ~ "\"release notes\""`.
- `?` matches one character and `*` any run of characters, inside a word but not at its start: `title ~ "rel*"`.
- A word followed by `~` is matched loosely (spelling variations): `text ~ "colour~"`.
- Wildcards and `~` must sit inside a quoted value.

## Ordering

- `ORDER BY field [ASC|DESC], field [ASC|DESC]` at the end of the query.
- Fields that can be ordered: `created`, `lastmodified`, `title`, `id`, `type`, `space`. Ordering by a field that can hold several values (such as `label`) gives `400`.
- Without `ORDER BY`, `GET /rest/api/search` orders by relevance and `GET /rest/api/content/search` orders by relevance too, falling back to `id` for ties.

## Open questions

- The published sources disagree on whether `parent` supports `IN` and `NOT IN`. Aerie supports them.
- The published model shows a search result's entity under a generic name; aerie writes it under `content` as clients expect. The lab should record the field name real servers use, and whether `/rest/api/search` returns `space` and `user` results (and which `type` values ask for them).
- The highlight marks used with `excerpt=highlight` are aerie's choice until the lab records what real servers send.
