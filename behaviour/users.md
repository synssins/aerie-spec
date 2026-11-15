# Users: Data Center compatible

Sources: L-0001, L-0003, L-0005, L-0007, L-0015

T1 covers reading users and the groups a user is in. Creating, changing and disabling users, passwords, settings and watches are T2 or T3. Shared rules are in `common.md`. How callers prove who they are is in `identity.md`.

## The user object

Wherever a user appears (`version.by`, `history.createdBy`, space `creator`, the user operations below), it has this shape:

- `type`: `known` (a real account), `anonymous` (no one signed in), or `unknown` (an account that no longer exists, kept so old versions still name someone).
- `username`: the login name. Absent for `anonymous`.
- `userKey`: a stable opaque key that does not change when the username changes. Aerie uses the user's own id as 32 lower-case hex digits (see `common.md`, Identifiers). Absent for `anonymous`.
- `displayName`: the full name to show.
- `profilePicture`: `path` (relative to `_links.base`), `width`, `height`, `isDefault` (true when the user has no uploaded picture).
- `_links.self`: the REST path of the user (`/rest/api/user?key=...`).
- Expandable: `status` (`current` or `deactivated`), `details.personal` and `details.business` (T2).

Aerie gives an `unknown` user the `userKey` it had, a `username` and `displayName` equal to that key, and the default picture. This is aerie's choice; the lab should record what real servers return for a deleted account.

## Reading a user

`GET /rest/api/user`: pick the user with `key` (user key) or `username`. Exactly one of the two is required, otherwise `400`. Unknown user: `404`. Anonymous callers get `403` when the site hides user profiles from anonymous users. Who may read another user is under Who may look people up.

`GET /rest/api/user/current`: the caller. For an unauthenticated request it returns the anonymous user, `200`.

`GET /rest/api/user/anonymous`: how the anonymous user looks: `type: anonymous`, a `displayName` (aerie uses `Anonymous`), the default picture.

## Listing users

`GET /rest/api/user/list`

- Every account on the site, enabled and disabled, offset-paged, default `limit` 100.
- Order: by username.
- Needs an authenticated caller who may look people up (below), otherwise `403`. Anonymous callers get `401`.

## Groups of a user

`GET /rest/api/user/memberof`

- Pick the user with `key` or `username` as above.
- Returns the groups the user is directly a member of, offset-paged, default `limit` 200, ordered by name. Each group is `{type: "group", name, _links}`.
- Unknown user: `404`. `403` when the caller may not see group memberships: only administrators may read another user's groups; anyone may read their own. Anonymous callers get `401`.

## Who may look people up

Question Q-0015. Aerie's own rule for looking people up applies:

- Reading another user and listing users: administrators, owners of any space, and anyone who may edit pages somewhere. Everyone may read their own record.
- Deactivated accounts are listed, with `status: deactivated` when expanded.
- Deleted accounts and aerie's emergency administrator account are never listed, and reading them gives `404`. A deleted account still appears as `type: unknown` where it made a version.
