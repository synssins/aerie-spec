# Spaces: Data Center compatible

Sources: L-0001, L-0002, L-0003, L-0004, L-0005, L-0006, L-0007, L-0013, L-0014, L-0015

Covers reading, listing, creating, updating, archiving, restoring and deleting spaces, listing a space's content and its trash. Space permissions, properties, categories, colour schemes and watchers are T2 or later. Shared rules are in `common.md`.

## The space object

- `id`: integer, unique on the site, never reused.
- `key`: the short identifier clients use in paths, unique on the site. Global space keys follow the key rules below (question Q-0001). Personal space keys are `~` followed by the owner's username (for example `~jdoe`).
- `name`: display name, up to 200 characters.
- `type`: `global` (a site space) or `personal` (owned by one user).
- `status`: `current` or `archived`.
- `_links`: `self`, `webui`, and `collection` on single reads.
- Expandable: `description` (holds `plain` and `view`, each `{value, representation}`; `plain` is what clients write), `icon` (`path`, `width`, `height`, `isDefault`), `metadata` (holds `labels`, the space's own labels), `creator`, `creationDate`, `lastModifier`, `lastModificationDate`, `retentionPolicy` (T3; aerie returns it unexpanded only), `permissions` (T2).

### Space keys

Aerie's own space keys are 2 to 32 characters of lower-case letters, digits and hyphens. Compatible clients send and expect upper-case or mixed-case keys. The compatible surface maps between the two:

- **Returned**: the aerie key in upper case. Aerie key `ops` is returned as `OPS`, and `team-a` as `TEAM-A`.
- **Accepted**: any case, in paths, query parameters, bodies and CQL. Aerie lower-cases a key before it looks it up, so `OPS`, `Ops` and `ops` name the same space.
- **Stored**: when a space is created through the compatible surface, aerie lower-cases the key, then applies its own key rules. A key that breaks them gives `400`; for example, a key of 33 or more characters, or one with an underscore. `LAB` is stored as `lab` and returned as `LAB`.
- **Compared**: always without regard to case. Two keys that differ only in case are the same key, so creating the second one gives `400`.
- A key is returned in upper case even when the client sent it in another case.

### No home page

Aerie spaces have no home page (question Q-0004). A space's top-level pages and folders are the nodes at the top of its tree.

- `homepage` is never returned and is not listed in `_expandable`. Asking for it with `expand=homepage` (or a deeper path such as `homepage.body.storage`) is ignored, as for any unknown expansion.
- Creating a space does not create a page.
- `homepage` in a create or update body is ignored.
- Clients that start from the home page to walk the tree should list the space's top level instead: `GET /rest/api/space/{spaceKey}/content/page?depth=root`.

## Reading one space

`GET /rest/api/space/{spaceKey}`: `404` when the key is unknown or the caller cannot view the space. The key in the path is matched without regard to case.

## Listing

`GET /rest/api/space`

- Filters: `spaceKey` (repeatable), `spaceKeys` (comma-separated, since 9.5.1), `spaceKeySingle` (one key), `spaceId` (repeatable, since 9.5.1) and `spaceIds` (comma-separated, since 9.5.1), `type` (`global` or `personal`), `status` (`current` or `archived`), `label` (repeatable; spaces carrying that space label), `contentLabel` (repeatable, since 9.5.1; spaces holding content with that label), `favourite` (when `true`, only the caller's favourite spaces), `hasRetentionPolicy` (accepted; aerie has no retention policies in T1, so `true` returns an empty list).
- Key and id filters cannot be combined: sending both gives `400`. All key filters together form one set.
- Without `status`, both current and archived spaces are listed.
- Order: by name, then key.
- Offset paging, default `limit` 25.

## Creating

`POST /rest/api/space` creates a global space. `POST /rest/api/space/_private` creates a global space only its creator can see.

- Who sees the new space (question Q-0012): `POST /rest/api/space` makes it readable by everyone signed in (aerie's Signed in visibility), the closest match to a site whose default space permissions let all users view. `POST /rest/api/space/_private` makes it Private: only its creator, people and groups later given a role, and administrators.

- Body: `key` (required), `name` (required), `description.plain.value` (optional, with `representation: plain`).
- Key rules as above; a bad key gives `400`. A key already in use, in any case, gives `400`.
- No page is created with the space. The creator gets full permissions on it.
- The caller needs the site permission to create spaces, otherwise `403`.
- Result: `200` with the space.

`POST /rest/api/space/personal` (since 9.5.1) creates the caller's personal space.

- Body: `description` (same shape as above) and `isPrivate` (boolean; the published model also lists `private` with the same meaning, so aerie reads either and `isPrivate` wins).
- `409` when the caller already has a personal space. `403` when the caller may not have one.
- In T1 only an administrator may create one; everyone else gets `403` (question Q-0010). The space is Private whatever `isPrivate` says. Its key is `~` followed by the owner's username in lower case, and its owner gets the Owner role. Personal spaces for every user, with visibility from `isPrivate`, are T2.
- Result: `200` with the space.

## Updating

`PUT /rest/api/space/{spaceKey}`

- Body: `name` (required), `key` (required; must match the path without regard to case, the key cannot be changed: `400`), `description.plain.value` (optional). `homepage` is ignored.
- Requires space administration permission, otherwise `403`.
- Result: `200` with the space.

## Archiving and restoring

`PUT /rest/api/space/{spaceKey}/archive` sets `status` to `archived`. `PUT /rest/api/space/{spaceKey}/restore` sets it back to `current`.

- Both are idempotent: archiving an archived space is a success that changes nothing.
- Archived spaces stay readable but are left out of search unless `includeArchivedSpaces=true` (see `search-cql.md`). Writes to content in an archived space give `403`.
- Requires space administration permission, otherwise `403`. Anonymous callers get `401`.
- Result: `204`.

## Deleting

`DELETE /rest/api/space/{spaceKey}`

- Starts deletion as a long task (see `common.md`) and returns `202` with a `LongTaskSubmission`. The space is gone (reads give `404`) as soon as the task is accepted, even though content clean-up continues in the background.
- Deletes everything in the space, including trash and attachments. It cannot be undone.
- Requires space administration permission, otherwise `403`.

## Content of a space

`GET /rest/api/space/{spaceKey}/content` returns a map keyed by type, with `page` and `blogpost`, each a paged list (like the `child` map in `content.md`). `GET /rest/api/space/{spaceKey}/content/{type}` returns one list directly; `type` is `page` or `blogpost`, anything else gives `400`.

- `depth`: `all` (default) for every item, `root` for top-level pages only (blog posts are always top-level).
- Offset paging, default `limit` 25. Only `current` content is listed.

## Trash

`GET /rest/api/space/{spaceKey}/trash` (since 9.5.1) lists the space's trashed content, newest-trashed first, with cursor paging (default `limit` 25). Needs the space's bin right in aerie (question Q-0014), otherwise `403`.

`DELETE /rest/api/space/{spaceKey}/trash` (since 9.5.1) purges everything in the space trash, with the meaning of purge in `content.md` (question Q-0016). Needs the space's bin right, otherwise `403`. Result: `204`.

## Open questions

- Aerie's keys may hold hyphens. No ledgered source says whether the vendor's keys may. A client that checks the characters of a returned key may reject `TEAM-A`. The lab should record whether common clients do.
- The `~username` form for personal spaces is aerie's choice; none of the sources read so far state it.

- Whether `GET /rest/api/space` without `status` returns archived spaces on real servers. Aerie returns both; to be confirmed by the lab.
