# Content: Data Center compatible

Sources: L-0001, L-0002, L-0003, L-0004, L-0005, L-0006, L-0007, L-0013, L-0014, L-0015

Covers pages, blog posts and comments through `/rest/api/content`: reading, listing, creating, updating, trashing and purging, versions and history, children, descendants, the cursor scan, and macro bodies. Attachments are content too, but have their own file (`attachments.md`). Shared rules (paging, expansion, links, errors) are in `common.md`. Body formats and the conversion operation are in `representations.md`.

## The content object

Every piece of content has:

- `id`: a string of decimal digits, unique across all content types on the site, never reused. How aerie assigns it, and the aerie UUID that is accepted in its place, are in `common.md` (Identifiers).
- `type`: `page`, `blogpost`, `comment` or `attachment`.
- `status`: `current` (live), `trashed` (in the space trash), `historical` (an older version, only seen when asking for a version), `draft` (an unpublished draft).
- `title`: plain text, up to 200 characters (question Q-0013). A longer title gives `400`. Comments have an empty title or a title derived from their parent; clients must not rely on it.
- Expandable fields: `space`, `history`, `version`, `ancestors`, `container`, `body`, `children`, `descendants`, `metadata`, `restrictions`, `operations`. Plus `extensions`, always present, holding type-specific values:
  - pages and blog posts: `position` (integer order among siblings, or absent when unordered),
  - comments: `location` (`footer`, `inline` or `resolved`),
  - attachments: see `attachments.md`.
- `_links`: `self`, `webui`, `tinyui`, and `edit` where the caller may edit.

Notes on the expandable fields:

- `space`: the space object (see `spaces.md`). Every page, blog post and comment belongs to exactly one space.
- `ancestors`: pages only. The chain of parent pages from the top of the space tree down to the direct parent, oldest first. A top-level page has an empty list. Each ancestor is a short content object (`id`, `type`, `status`, `title`, `_links`).
- `container`: for a comment, the page or blog post it is on; for a page or blog post, its space.
- `version`: the current version (see Versions below).
- `history`: creation and version links (see History below).
- `metadata`: `labels` (a paged label list, see `labels-properties.md`) and, for attachments, `mediaType` and `comment`. Other keys are T2.
- `operations`: what the caller may do, as a list of `{operation, targetType}` pairs, where `operation` is one of `read`, `update`, `delete`, `create` and `targetType` names the content type.
- `restrictions`: read and update restriction summaries. Full restriction handling is T2 (`restrictions.md`); in T1, aerie returns the expandable keys `read` and `update` with empty restriction lists when the content has none.

## How the page tree maps onto aerie's tree

Question Q-0002. Aerie's tree has two kinds of node. A **folder** holds other nodes. A **page** holds none. Compatible clients expect any page to have child pages, so the surface maps the two like this:

- Folders and pages both appear as `type: page`. Nothing on the compatible surface tells them apart.
- A folder's body is its own content. A folder with no text has an empty storage body (`""`).
- `extensions.position` is the node's place among its siblings in aerie's tree.
- Children (`child`, `child/page`): for a folder, the nodes directly inside it, in aerie's sibling order. For a page, an empty list.
- `ancestors`: the folders above the node, top first. A node at the top of the space has an empty list.
- Descendants (`descendant/page` and its count): every node below a folder, depth-first in sibling order. A page has none.
- CQL `parent` and `ancestor` follow the same mapping.
- New content is created as an aerie page.
- **A page that gains a child becomes a folder.** This happens when a page is created or moved with `ancestors` naming an aerie page. That page becomes a folder in place, then the new child goes inside it. It keeps its id, title, body, versions, labels and attachments. The change is recorded as an edit by the caller.
- A folder stays a folder even when its last child is removed or moved out.

## Reading one item

`GET /rest/api/content/{id}`

- Default expansion: `space`, `history`, `version`.
- `status` (repeatable): which statuses to accept. Default `current`. If the content is in another status and that status is not listed, the response is `404`. Pass `status=trashed` to read trashed content, `status=any` for current or trashed. Trashed content is visible only to callers who hold the space's bin right (question Q-0014); for anyone else it does not exist (`404`).
- `version`: return that historical version instead of the current one. The response carries `status: historical` unless the number asked for is the current version. A version that does not exist gives `404`.
- Draft content is readable only by its author (and by others it is shared with, T2).

## Listing

`GET /rest/api/content`

- Filters: `type` (`page` or `blogpost`, default `page`), `spaceKey`, `title` (exact match, case-insensitive), `status` (repeatable, default `current`, `any` allowed), `postingDay` (blog posts only, `yyyy-mm-dd`, the day the post is dated), `ids` (repeatable, fetch these ids; when present, `type` is not applied).
- Order: by `id` ascending, which is creation order.
- Offset paging, default `limit` 25. No default expansion.
- Comments and attachments are not listed here; use `child` (below) or search.
- Unknown `type` gives `400`.

## Scan

`GET /rest/api/content/scan` (since 9.0.1; `type` filter since 9.5.1)

- Walks all content the caller can see in a stable order (by id), using cursor paging (see `common.md`). Meant for exporters that need every item once.
- Filters: `spaceKey`, `type` (`page`, `blogpost`, `comment`; default `page`; only read on the first request, later requests take it from the cursor), `status` (repeatable, default `current`, `any` allowed). `title` and offset paging are not supported here.
- Default `limit` 25.

## Creating

`POST /rest/api/content`

Body fields aerie reads:

- `type` (required): `page`, `blogpost` or `comment`. `attachment` gives `400`; attachments are created through the child attachment operation.
- `title`: required for pages and blog posts. Two pages in one space may not share a title (case-insensitive): the second create gives `400`. Blog posts must have a unique title within their posting day and space.
- `space.key`: required for pages and blog posts. Unknown space gives `404`. For comments, the space is taken from the container.
- `ancestors`: optional, pages only. Only the last entry's `id` matters: it becomes the parent. The parent must be a `current` page in the same space, otherwise `400`. If the parent is an aerie page rather than a folder, it becomes a folder first (see the tree mapping above). Without ancestors the page is created at the top of the space tree.
- `container`: required for comments: `{ "id": ..., "type": "page" | "blogpost" }`. To reply to a comment, set `ancestors` to the parent comment.
- `body.storage.value` with `body.storage.representation: storage`, or `body.editor` with `representation: editor`. Other representations give `400`. Missing body means an empty body. See `representations.md` for what aerie accepts and stores.
- `status`: `current` (default) or `draft`. The `status` query parameter means the same thing and wins if both are given.
- `id`: when given together with `status: current` and the id names an existing draft, the draft is published as a new current item (its body and title come from the request when given).
- `extensions.location` (comments only): `footer` (default) or `inline`. Inline comments need T2 anchoring and are stored as footer comments in T1.

Result: `200` with the created content, expanded as asked by `expand` (default `space,history,version`). The new item has `version.number` 1.

Errors: `400` for missing or invalid fields, `403` when the caller may not add content in that space, `404` for an unknown space, parent or container.

## Updating

`PUT /rest/api/content/{contentId}`

- The body is a content object. `type` must match the stored type, otherwise `400`. `version.number` is required and must be exactly the current number plus one.
- If `version.number` is not current + 1, the response is `409` and nothing changes, unless `conflictPolicy` says otherwise. The only value aerie supports is `abort` (the default); any other value gives `400`.
- Fields that may change: `title`, `body` (pages, blog posts, comments), `ancestors` (moves a page under a new parent in the same space, turning a parent page into a folder as above; moving across spaces gives `400` in T1), `status` (see below), `version.message` (a change note), `version.minorEdit` (when `true`, watchers are not notified).
- Fields left out of the body are left unchanged; `body` left out keeps the current body.
- Each successful update creates a new version and keeps the old one as `historical`.
- `status` query parameter: the status the content has now. Default `current`. Use `status=trashed` with a body `status: current` to restore a trashed item; restoring does not create a new version.
- `asyncReconciliation` (since 9.2.4): accepted and ignored. Aerie always reconciles at once.
- Changing a page title to one already used in the space gives `400`.

Result: `200` with the updated content.

## Trashing and purging

`DELETE /rest/api/content/{id}`

- A `current` page, blog post or comment is moved to the space trash: its status becomes `trashed`. Child pages are not trashed; they move up to the trashed page's parent. Comments on it stay with it.
- A `trashed` item is purged when the request also carries `status=trashed`. Without that parameter, deleting a trashed item gives `404`. Purging, restoring and reading trashed items need the space's bin right (question Q-0014), otherwise `403`.
- What purged means (question Q-0016): the item leaves the space trash and every read of the compatible surface for good, and its id is never used again. Aerie's administrators' bin keeps it until an administrator empties that bin; that is the step that destroys the data. The purge is written to aerie's audit log.
- Comments are never trashed: deleting one removes it for good, with its replies.
- A `draft` is deleted for good when the request carries `status=draft`. Aerie has no unpublished drafts in T1, so this gives `404` (see What T1 leaves out).
- Result: `204` with no body. (The published spec also lists `200`; aerie always uses `204`.)
- `409` when the item cannot be removed in its current state, for example a page that another operation has locked.

## Versions

Each page, blog post and comment has a version chain starting at 1.

The `version` object: `number` (integer), `when` (ISO 8601 date-time, UTC), `message` (the change note, may be empty), `minorEdit` (boolean), `by` (the user who made the version, see `users.md`), `hidden` (boolean, true only for versions the system made), `_links`, and the expandable `content` (the content as it was at that version).

`DELETE /rest/api/content/{id}/version/{versionNumber}`

- Removes one historical version of a page or blog post. The current version cannot be removed: asking gives `400`. Comments do not support this: `400`.
- Version numbers are not reused or shifted. After removing version 3 of 5, the remaining versions keep numbers 1, 2, 4, 5.
- A removed version is gone from the compatible surface: it is not listed and cannot be read with `version=` (`404`). Aerie's own history, which only grows, keeps it, and the removal is written to the audit log (question Q-0016).
- The caller needs permission to edit the content, otherwise `403`.
- Result: `204`.

## History

`GET /rest/api/content/{id}/history`

Returns:

- `latest` (boolean): true when this is the current version.
- `createdBy` (user) and `createdDate` (date-time): who made version 1 and when.
- Expandable: `lastUpdated` (the current version object), `previousVersion` and `nextVersion` (relative to the version the history belongs to; absent at the ends of the chain), `contributors` (`publishers.users`: the users who made versions, and `publishers.userKeys`).
- `_links`.

## Children

`GET /rest/api/content/{id}/child`

- Returns a map, not a list. Keys are content types; each value is a paged list of direct children of that type. The types included are chosen with `expand`: `expand=page,comment,attachment`. Without `expand`, every type is listed as an `_expandable` key and no children are returned.
- Nested expansion works per type: `expand=page.version`.
- `parentVersion`: when not 0, the children as they were at that version of the parent. Aerie supports `0` (now) only in T1; any other value gives `400`.
- Children of a page: `page` (child pages, in sibling order), `comment` (footer comments), `attachment`. A blog post has no `page` children. A comment's children are its reply comments.

`GET /rest/api/content/{id}/child/{type}` returns one of those lists directly, as an offset-paged response, default `limit` 25. Unknown `type` gives `400`.

`GET /rest/api/content/{id}/child/comment` returns comments with extra filters:

- `depth`: empty (default) for top-level comments only, `all` for every comment in the thread, flattened, in creation order.
- `location` (repeatable): `footer`, `inline`, `resolved`. Empty means all.

Child pages are ordered by `extensions.position` when set, then by title; other types by creation time.

## Descendants

`GET /rest/api/content/{id}/descendant`

- Like `child`, a map by type, picked with `expand`, but covering all levels. The published behaviour supports only `comment` descendants (all comments in the thread) in this map; aerie also supports `page` (all pages under it, depth-first in tree order) and `attachment` (attachments on the content itself only).

`GET /rest/api/content/{id}/descendant/{type}`: one list, offset-paged, default `limit` 25.

`GET /rest/api/content/{id}/descendant/page/count` (since 10.2.20)

- Returns `{ "count": n }`: the number of `current` pages anywhere below the page. The page itself is not counted.
- `404` when the content is not a page or the caller cannot view it.

## Macro bodies

`GET /rest/api/content/{id}/history/{version}/macro/id/{macroId}` returns the macro with that `ac:macro-id` in the given version's storage body as `{ "name", "body", "parameters" }`: the macro name, its body in storage format (empty string when it has none), and its parameters as a string-to-string map. `404` when no macro has that id.

`GET /rest/api/content/{id}/history/{version}/macro/hash/{hash}` is the older lookup by a render-time hash. Aerie computes the hash as the lowercase hex SHA-256 of the macro's storage markup and accepts it; clients that got a hash from a vendor renderer will get `404`. This is a known gap.

## What T1 leaves out

Blog posts, comments and unpublished drafts (question Q-0007). Aerie holds none of them yet; each needs a new aerie data model. Until then the operations that touch them answer as follows:

- Lists of blog posts and of comments are empty (`size: 0`), alone and inside `child`, `descendant` and space content maps.
- Creating a blog post or a comment gives `400`. So does `status: draft` (or `status=draft`) on create.
- Reading, updating or deleting with `status=draft` finds nothing: `404`.
- Blog posts and comments are T2. Unpublished drafts are T3: an aerie draft is someone's unsaved edits to an existing page, not a new page, so the vendor's model maps onto it only with collaborative editing.

Also left out:

- Bulk page operations (`/rest/api/content/bulk/...`, since 10.2.20): T2.
- Blueprint draft publishing (`/rest/api/content/blueprint/instance/{draftId}`): T3.
- Content properties, restrictions, watchers: T2 (`labels-properties.md`, `restrictions.md`).

## Open questions

- The published spec says the descendant page count is a JSON object but does not name its field. Aerie uses `count` until the lab records what a real server returns.
- Whether a trashed page's child pages move up to its parent or stay under it in the trash. Aerie moves them up; to be confirmed by the lab.
- Whether title uniqueness on create is case-insensitive on real servers. Aerie treats it as case-insensitive; to be confirmed by the lab.
