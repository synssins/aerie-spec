# Q-0029: Who sees negatives, and what ratings feed

Raised by: the spec room, while specifying the owner's request for page ratings.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

Negative ratings are visible only to Supervisors and higher in the page's space. Do site administrators count as higher? Do ratings feed search ranking or send notifications?

## Answer

- Only the space's Supervisors and Owners see negative counts, who rated, when, and reasons. A site administrator without one of those roles sees what a reader sees. The owner wrote "and nobody else"; this ruling reads it strictly.
- Readers see the positive count and the solved count only: no total, no average, no score, so negatives cannot be worked out from what they see.
- The audit log records that someone rated a page, never the value or the reason, so it does not reveal negatives to administrators either.
- Ratings do not change search ranking or ordering. Negative ratings feeding ranking would let readers infer them, and ratings could be used to bury pages. They send no notifications and are not sent to AI features.

Left to the owner: whether site administrators should see negatives everywhere, and whether ratings should later feed ranking. This ruling says no to both.

Written in `behaviour/ratings.md` (Who sees what, What ratings do not do, Audit).
