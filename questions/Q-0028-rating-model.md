# Q-0028: What a rating holds

Raised by: the spec room, while specifying the owner's request for page ratings.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

How do "This solved my problem", the negative reason, and changing a rating fit together?

## Answer

- One rating per person per page, with `value` `positive` or `negative`.
- "This solved my problem" is a positive rating with `solved: true`; pressing it sets both. `solved` with a negative rating is `400`.
- The reason is optional, only on negative ratings, plain text, at most 280 characters. A reason on a positive rating is `400`.
- Changing a rating replaces it; removing deletes it for good. Ratings survive page edits. Each rating records the page version it was last saved against, so Supervisors can tell whether a negative rating predates a fix.
- Anyone who can read the page may rate it, including its own editors; one vote each.
- Readers see their own rating, including their own negative and reason, so they can change it.
- At most 30 rating changes per person per minute.

Written in `behaviour/ratings.md` (A rating, Calls).
