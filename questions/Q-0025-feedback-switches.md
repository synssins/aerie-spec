# Q-0025: The three-level switch for comments and ratings

Raised by: the spec room, while specifying the owner's request for page comments and page ratings.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

Comments and ratings switch on and off at the site, the space and the page, the most specific setting wins, and "off" above cannot be overridden below. What are the defaults, who sets each level, and what happens to existing comments and ratings when a level is off?

## Answer

- Each level is a plain on/off value per feature, and the effective value is on only if all three are on. Because "on" at a lower level can never override "off" above, a lower level can only switch off, so no third "inherit" value is needed.
- Site: `commentsEnabled` and `ratingsEnabled` in new feedback settings, set by Admins. Space: the existing `commentsEnabled` on the space, plus a new `ratingsEnabled`, set by whoever may change space settings today. Page: both values in the page's feedback settings, set by the space's Supervisors and Owners (the moderators).
- Defaults: on at every level. The existing space `commentsEnabled` keeps its current values.
- Off does not delete anything. Readers no longer see the comments or rating counts, nobody can add or change any, and Supervisors and Owners still see and moderate everything. People can still delete their own comments and remove their own ratings. Switching back on shows what was kept.

For the owner: the defaults (on) are this ruling's choice. If comments or ratings should start off on a new install, only the site default changes.

Written in `behaviour/comments.md` (Switching comments on and off), `behaviour/ratings.md` (Switching ratings on and off); `feedback.settings.*`, `page.feedback.*` and the space field `ratingsEnabled` in `contracts/aerie/openapi.yaml`.
