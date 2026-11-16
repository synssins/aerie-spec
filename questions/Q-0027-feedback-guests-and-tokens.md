# Q-0027: Guests and API tokens for comments and ratings

Raised by: the spec room, while specifying the owner's request for page comments and page ratings.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

Guests who are not signed in never comment or rate. May they read comments or see rating counts on pages in spaces readable by anyone? And may API tokens use the new calls?

## Answer

- Guests see neither comments nor ratings: every comment and rating call is `401` without a session. Comments name the people who wrote them, and a public page should not publish staff names and discussions by default. This is the simple, safe choice; it can be loosened later.
- API tokens may read the comment list (`x-kb-api-token: read`), like page reads. Every other new call is session only (`closed`): no posting, moderating, rating, or reading ratings from scripts. This keeps bulk posting and vote stuffing out, and keeps who-rated-what inside the web app.

Written in `behaviour/comments.md` (Who may do what), `behaviour/ratings.md` (Who sees what); the `x-kb-api-token` value of each new call.
