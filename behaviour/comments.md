# Page comments

Sources: L-0025

Aerie's own feature, not a vendor-compatible surface. Signed-in people who can read a page comment on it; the space's Supervisors and Owners moderate. Every rule here is aerie's own design. The decisions behind the details are questions Q-0022 to Q-0027.

The calls are in `contracts/aerie/openapi.yaml` under `/api/v1`, marked `x-aerie-status: planned`. They follow the as-is conventions: problem bodies (RFC 9457) for refusals, `401` when not signed in, `403` when not allowed, `404` for anything the caller cannot see, `422` for a request refused by a rule, `429` for too many calls. Each call runs through the action pipeline under its `x-kb-action` name.

The Data Center compatible surface is not changed by this file. Its comment operations still answer as question Q-0007 says (empty lists, `400` on create). Exposing these comments there is T2 work; the model below is built so that it can be (one level of replies maps to the compatible reply model).

## Who may do what

| Who | Read comments | Comment and reply | Edit or delete own | Hide, delete, restore any |
|---|---|---|---|---|
| Not signed in (guest) | No | No | No | No |
| Signed in, can read the page (any space role, or none when the space is readable by everyone signed in) | Yes | Yes, when comments are on | Yes (rules below) | No |
| Supervisor or Owner of the page's space | Yes, including hidden and deleted | Yes, when comments are on | Yes | Yes |

- "Can read the page" is the same check page reading uses, including any later page restrictions. Someone who cannot read the page gets `404` from every comment call on it, never `403`.
- Guests get `401` from every comment call, also on pages in spaces readable by anyone. Guests do not see comments at all (question Q-0027).
- Moderation needs the Supervisor or Owner role in that space. Being a site administrator is not enough on its own (question Q-0024).
- API tokens: reading the list (`comment.list`) is open to tokens with read access (`x-kb-api-token: read`). Every other comment call is session only (`closed`), so scripts cannot post or moderate (question Q-0027). A valid API token on a session-only call gets `403`; an unknown, expired or revoked token gets `401` (question Q-0039).

## Switching comments on and off

Comments are switched at three levels. The effective setting for a page is on only if all three are on (question Q-0025):

1. The site: `commentsEnabled` in the feedback settings (`feedback.settings.read` / `feedback.settings.save`, Admins only). Default on.
2. The space: the existing `commentsEnabled` on the space (`SpaceDto`, `UpdateSpaceRequest`), changed by whoever may change space settings today. Default on. Today the field is stored, returned and editable but nothing acts on it, so using it as the switch changes nothing for existing data.
3. The page: `commentsEnabled` in the page's feedback settings (`page.feedback.read` / `page.feedback.save`). Changed by the space's Supervisors and Owners. Default on.

"Off" at a higher level cannot be overridden below: a page set on in a space set off has comments off. Each level only ever switches comments off for what is under it.

When comments are off for a page:

- Nobody can comment or reply: `422`.
- Existing comments are kept but not shown to anyone except the space's Supervisors and Owners, who can still moderate them. The list answers with `enabled: false` and, for everyone else, no comments.
- Authors can still delete their own comments (so a person can always remove what they wrote).
- Switching back on shows the visible comments again, unchanged.

The same switches, with `ratingsEnabled`, control ratings (`ratings.md`).

## Comments

A comment (`CommentDto`) has:

- `id` (uuid), `pageId`, `parentId` (null for a top-level comment).
- `author`: the person, as `{type: "user", id, name}`. Shown to everyone who can see the comment.
- `text`: plain text, 1 to 4000 characters after trimming, line breaks kept (question Q-0023). No HTML, no markup. Clients show it escaped; they may turn `http`/`https` URLs into links with `rel="nofollow noopener ugc"`. Text that is empty after trimming, or longer than 4000 characters, gives `400`.
- `createdAt`, `editedAt` (null if never edited).
- `state`: `visible`, `hidden` or `deleted`.
- `canEdit`, `canDelete`, `canModerate`: what the caller may do with it.
- `replies`: on top-level comments only, their replies in creation order.

### Replies

One level of replies (question Q-0022). A reply's `parentId` must be a top-level comment on the same page whose state is `visible`; otherwise `400`. A reply to a reply is `400`; clients reply to the top-level comment instead. A top-level comment has at most 100 replies; the 101st is `422`.

### Listing

`GET /api/v1/pages/{id}/comments` (`comment.list`):

- Top-level comments, oldest first, paged with `limit` (default 50, at most 200; more is `400`) and `offset`, with `total`. Each carries all its replies.
- The answer also says `enabled` (the effective switch), `canComment` and `canModerate` for the caller.
- Readers see `visible` comments, plus their own `hidden` comments (marked hidden, so they know). A `hidden` or `deleted` comment that still has visible replies is shown to readers as a placeholder: `state` set, `author` and `text` null, so the replies keep their context.
- Supervisors and Owners see every comment, with its state, author and text, except comments their own authors deleted (those are gone).

### Posting

`POST /api/v1/pages/{id}/comments` (`comment.create`) with `{text, parentId?}` answers `201` with the comment.

- Comments are on pages, not folders: a folder gives `422`.
- Comments off: `422`.
- At most 10 comments per person per minute across the site; more gives `429` with `Retry-After`.

### Editing

`PUT /api/v1/comments/{id}` (`comment.edit`) with `{text}` (question Q-0023):

- Only the author, and only within 15 minutes of `createdAt`. Anyone else gets `403`; the author after 15 minutes gets `422`.
- Only `visible` comments can be edited; others give `422`.
- Sets `editedAt`. Earlier text is not kept.
- Moderators cannot change other people's text; they hide or delete instead.

### Deleting your own

`DELETE /api/v1/comments/{id}` (`comment.delete`) by the author, at any time, also while comments are off:

- The text is removed at once and for good. It cannot be restored, not even by moderators (question Q-0024).
- If the comment has replies, a placeholder stays so the replies keep their place; otherwise nothing is left.
- Answers `204`.

## Moderation

By the space's Supervisors and Owners (question Q-0024):

- **Hide** (`POST /api/v1/comments/{id}/hide`, `comment.hide`): the comment stays, but only moderators and its author see it. Reversible at any time. `204`.
- **Delete** (`DELETE /api/v1/comments/{id}`, `comment.delete`, by a moderator on someone else's comment): state `deleted`. Only moderators see it, in the list, until it is purged 30 days later. `204`.
- **Restore** (`POST /api/v1/comments/{id}/restore`, `comment.restore`): a hidden comment, or a comment a moderator deleted and not yet purged, becomes `visible` again. A comment its author deleted cannot be restored: `404`. Restoring a `visible` comment is `409`. `204`.
- Hiding or deleting a top-level comment does not touch its replies; they stay as they are, under the placeholder.
- Hiding an already hidden comment, or deleting an already deleted one, is `204` and changes nothing.

## Pages that move, are trashed or are purged

- Comments belong to the page. Moving a page to another space moves its comments; from then on the new space's roles and switches apply.
- A trashed page's comments are kept with it and come back when it is restored. Purging the page purges its comments.
- Restoring an old page version does not change comments.

## Notifications

Question Q-0026. Two new notification kinds, each with the usual per-person preference (`UserSettings.notify`), default in-app on and email off:

- `comment.page`: a new top-level comment on a page you own (the page's `details.owner`, when it is a user).
- `comment.reply`: a reply to your comment.

Nobody is notified of their own comment, nobody is notified about a page they cannot read, and nobody is notified about hidden or deleted comments. A person who is both owner and replied-to gets one notification. Group owners are not notified. There are no mentions in this first version.

## Audit

Every call that changes data writes an audit entry: action (the `x-kb-action` name), actor, time, `via`, `token`; target type `comment` and the comment id, or `page` and the page id for `page.feedback.save`. The comment text is never written to the audit log or the application log.

## Lab cases

Case names are suggestions; the lab owns its suite. The lab needs a space where it controls roles, a page, and people with Reader, Contributor, Supervisor and Owner roles, plus a second person who is a Reader.

- `comments.guest.refused`: no session: list and create on a page in a space readable by anyone give `401`.
- `comments.reader.create`: a Reader posts; `201`, `state: visible`, author is the Reader.
- `comments.unreadable.hidden`: a person who cannot read the page gets `404` on list and create.
- `comments.folder.refused`: creating on a folder gives `422`.
- `comments.text.limits`: empty after trimming, and 4001 characters, each give `400`; 4000 characters is `201`.
- `comments.reply.one_level`: replying to a top-level comment is `201`; replying to that reply is `400`.
- `comments.edit.window`: the author edits within 15 minutes (`200`, `editedAt` set); another person editing gets `403`; with the clock moved past 15 minutes the author gets `422`.
- `comments.delete.own`: the author deletes; `204`; the comment is gone from every list, including a Supervisor's; restore gives `404`.
- `comments.delete.placeholder`: a top-level comment with a reply is deleted by its author; readers see a placeholder with null `author` and `text`, and the reply.
- `comments.moderate.reader_refused`: a Reader or Contributor hiding someone else's comment gets `403`.
- `comments.moderate.admin_without_role`: a site administrator with no role in the space hiding a comment gets `403`.
- `comments.hide.restore`: a Supervisor hides; the other Reader no longer sees it; its author sees it as `hidden`; the Supervisor restores; everyone sees it again.
- `comments.moderator_delete.restore`: an Owner deletes someone's comment; only Supervisors and Owners see it as `deleted`; restore brings it back.
- `comments.moderator_delete.purge`: 30 days after a moderator delete (clock moved), the comment is gone and restore is `404`.
- `comments.switch.site_off`, `comments.switch.space_off`, `comments.switch.page_off`: each off gives `422` on create and an empty list with `enabled: false` for a Reader; a Supervisor still sees the comments.
- `comments.switch.no_override`: space off, page on: create is still `422`.
- `comments.switch.page_by_supervisor`: a Contributor setting the page switch gets `403`; a Supervisor gets `200`.
- `comments.switch.own_delete_when_off`: with comments off, the author can still delete their comment.
- `comments.rate_limit`: the 11th comment within a minute is `429` with `Retry-After`.
- `comments.token.read_only`: an API token lists comments (`200`) but cannot post (`403`); an expired token gets `401` on both.
- `comments.notify.owner_and_reply`: a comment notifies the page owner; a reply notifies the comment's author; neither notifies the person who wrote it.
- `comments.audit.no_text`: after create, edit, hide, delete and restore, audit entries exist for each, and no entry contains the comment text.
- `comments.page.move`: after the page moves to a space where the commenter has no access, the commenter gets `404`.

## Build slices

One pull request each, in this order. Slice F is shared with ratings and is built first.

1. **F: Feedback switches.** Site settings (`feedback.settings.read` / `save`), the page switches (`page.feedback.read` / `save`), the effective-setting function used by comments and ratings, the new space field `ratingsEnabled` next to the existing `commentsEnabled`, and their admin and space settings controls. Lab: the `*.switch.*` cases that need only the settings answers (`page.feedback.read` shows `effective`).
2. **C1: Comments.** Storage, list, post, reply, edit window, author delete with placeholders, limits, rate limit, page move/trash/purge behaviour, audit, and the comments section on the page in the web app. Lab: every case above except moderation, purge and notifications.
3. **C2: Moderation.** Hide, moderator delete, restore, the 30-day purge job, the moderator view in the web app. Lab: the `comments.moderate.*`, `comments.hide.*`, `comments.moderator_delete.*` cases.
4. **C3: Notifications.** The two notification kinds, preferences, and their place in notification settings. Lab: `comments.notify.*`.
