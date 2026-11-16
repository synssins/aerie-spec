# Q-0026: Who is notified about comments

Raised by: the spec room, while specifying the owner's request for page comments.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

Should page owners, or anyone else, be notified about new comments?

## Answer

Two new notification kinds, each with the usual per-person preference, default in-app on and email off:

- `comment.page`: a new top-level comment on a page you own (the page's owner in its details, when that owner is a user).
- `comment.reply`: a reply to your comment.

Nobody is notified of their own comment, of a hidden or deleted comment, or about a page they cannot read. One notification per person per comment. Pages owned by a group notify nobody, so one comment never fans out to a whole group. No mentions in this first version, and moderators are not notified (they would be notified of every comment in their spaces).

Written in `behaviour/comments.md` (Notifications).
