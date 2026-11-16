# Page ratings

Sources: L-0025

Aerie's own feature, not a vendor-compatible surface. A signed-in reader rates a page positive or negative; a positive rating can also say "This solved my problem". Negative ratings are seen only by the space's Supervisors and Owners. Every rule here is aerie's own design. The decisions behind the details are questions Q-0025, Q-0027, Q-0028 and Q-0029.

The calls are in `contracts/aerie/openapi.yaml` under `/api/v1`, marked `x-aerie-status: planned`, with the same conventions as `comments.md`: problem bodies, `401` not signed in, `403` not allowed, `404` not visible, `422` refused by a rule, `429` too many calls, and each call runs through the action pipeline under its `x-kb-action` name.

## A rating

One rating per person per page (question Q-0028). It has:

- `value`: `positive` or `negative`.
- `solved`: `true` when the person pressed "This solved my problem". Only with `positive`; `solved: true` with `negative` is `400`. Pressing the button sets `value: positive` and `solved: true` in one call.
- `reason`: optional, only with `negative`, plain text, at most 280 characters after trimming. A reason with `positive` is `400`; longer is `400`. Empty after trimming is stored as no reason.
- `version`: the page version the person was reading when they last saved the rating. Set by aerie from the current version; clients do not send it.
- `createdAt`, `updatedAt`.

Changing a rating replaces it (`updatedAt` changes, `createdAt` stays). Removing it deletes it for good. Ratings stay when the page is edited.

## Who sees what

| Who | Positive count and solved count | Own rating | Negative count | Who rated what, when, and reasons |
|---|---|---|---|---|
| Not signed in (guest) | No | - | No | No |
| Signed in, can read the page (Reader, Contributor, or no role) | Yes | Yes, including their own negative and reason | No | No |
| Supervisor or Owner of the page's space | Yes | Yes | Yes | Yes |

- "Nobody else" is strict (question Q-0029): a site administrator without the Supervisor or Owner role in that space sees what a reader sees.
- Nothing a reader can see reveals negatives: there is no total count, no average, and no score. Positive count and solved count only.
- Someone who cannot read the page gets `404` from every rating call on it. Guests get `401` (question Q-0027).
- All rating calls are session only (`x-kb-api-token: closed`), so scripts cannot rate or read ratings (question Q-0027).

## Switching ratings on and off

Same three levels as comments (question Q-0025): `ratingsEnabled` in the site feedback settings (Admins), on the space (new field next to `commentsEnabled`, changed by whoever may change space settings), and in the page's feedback settings (Supervisors and Owners). Effective only if all three are on; off above cannot be overridden below. Defaults: on.

When ratings are off for a page:

- Nobody can rate or change a rating: `422`.
- People can still remove their own rating.
- Readers see no counts (`enabled: false`, counts `0`). Supervisors and Owners still see everything.
- Switching back on shows the kept ratings again.

## Calls

- `GET /api/v1/pages/{id}/ratings` (`rating.summary`): `enabled`, `canRate`, `canSeeAll`, `positive`, `solved`, `mine` (the caller's rating or null), and `negative` (the count) only when `canSeeAll` is true; otherwise `negative` is absent.
- `PUT /api/v1/pages/{id}/ratings/mine` (`rating.save`) with `{value, solved?, reason?}`: creates or replaces the caller's rating; answers `200` with the summary. Folders give `422`. At most 30 rating changes per person per minute across the site; more is `429` with `Retry-After`.
- `DELETE /api/v1/pages/{id}/ratings/mine` (`rating.remove`): `204`, also when the caller had no rating.
- `GET /api/v1/pages/{id}/ratings/all` (`rating.list`): Supervisors and Owners only, others `403`. Every rating, newest change first, paged with `limit` (default 50, at most 200) and `offset`, with `total`. Optional `value` filter (`positive` or `negative`). Each entry: the person (`{type: "user", id, name}`), `value`, `solved`, `reason`, `version`, `createdAt`, `updatedAt`.

## What ratings do not do

Question Q-0029:

- They do not change search ranking, page order, or anything else a reader sees.
- They send no notifications.
- They are not sent to AI features.

## Pages that move, are trashed or are purged

As for comments: ratings move with the page and then follow the new space's roles and switches; a trashed page keeps its ratings; purging the page purges them.

## Audit

`rating.save` and `rating.remove` write audit entries (action, actor, time, `via`, target type `page` and the page id). The value and the reason are never written to the audit log or the application log, so the audit log does not show negatives to anyone who could not see them on the page.

## Lab cases

Same set-up as `comments.md`: a space with a Reader, a Contributor, a Supervisor and an Owner, a second Reader, and a site administrator with no role in the space.

- `ratings.guest.refused`: no session: summary and save give `401`.
- `ratings.positive`: a Reader saves `positive`; the summary shows `positive: 1`, `solved: 0`, `mine.value: positive`, and no `negative` field.
- `ratings.solved`: a Reader saves `{value: positive, solved: true}`; `positive: 1`, `solved: 1`.
- `ratings.solved_with_negative.refused`: `{value: negative, solved: true}` is `400`.
- `ratings.reason.rules`: a reason with `positive` is `400`; 281 characters with `negative` is `400`; 280 is `200`.
- `ratings.negative.hidden_from_readers`: the first Reader rates negative with a reason. The second Reader and the Contributor see `positive: 0` and no `negative` field; the first Reader sees their own negative and reason in `mine`.
- `ratings.negative.visible_to_supervisors`: the Supervisor's summary has `negative: 1`; `rating.list` shows the Reader, the value, the reason, the version and the times.
- `ratings.list.readers_refused`: the Reader and the Contributor get `403` on `rating.list`.
- `ratings.list.admin_without_role`: the site administrator with no role in the space gets `403` on `rating.list` and no `negative` field in the summary.
- `ratings.one_per_person`: saving twice leaves one rating; changing negative to positive moves the counts.
- `ratings.remove`: removing gives `204` and the counts drop; removing again gives `204`.
- `ratings.switch.page_off`: with the page switch off, save is `422`, a Reader sees `enabled: false` and zero counts, removing one's own rating still works, and the Supervisor still sees everything.
- `ratings.switch.no_override`: site off, space and page on: save is `422`.
- `ratings.unreadable.hidden`: a person who cannot read the page gets `404`.
- `ratings.folder.refused`: rating a folder is `422`.
- `ratings.token.refused`: an API token gets `401` on every rating call.
- `ratings.search.unchanged`: search results and their order for a fixed query are the same before and after ten negative ratings on the top hit.
- `ratings.audit.no_value`: audit entries exist for save and remove and contain neither the value nor the reason.

## Build slices

One pull request each, after slice F (feedback switches) in `comments.md`:

1. **R1: Ratings.** Storage, summary, save, remove, the rules above, rate limit, page move/trash/purge, audit, and the rating control on the page in the web app (positive, negative with optional reason, "This solved my problem"). Lab: every case except those on `rating.list`.
2. **R2: Supervisor view.** `rating.list`, the negative count in the summary, and a ratings panel on the page for Supervisors and Owners. Lab: `ratings.negative.visible_to_supervisors`, `ratings.list.*`.
