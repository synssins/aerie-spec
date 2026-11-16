# Q-0023: Comment format, length and editing

Raised by: the spec room, while specifying the owner's request for page comments.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

What may a comment contain, how long may it be, and can its author edit or delete it?

## Answer

- Plain text only, 1 to 4000 characters after trimming, line breaks kept. No HTML and no markup, so nothing in a comment can run in a reader's browser. Clients may turn `http`/`https` URLs into links marked `rel="nofollow noopener ugc"`.
- The author may edit a comment for 15 minutes after posting, to fix mistakes. After that it is fixed (`422`). Earlier text is not kept; `editedAt` shows that it changed. A short window means nobody can change what a comment said after others have replied to it, and aerie needs no edit history.
- The author may delete their own comment at any time, also while comments are switched off. The text is removed for good. If the comment has replies, a placeholder with no author and no text keeps their place.
- Moderators never edit other people's text; they hide or delete.

Written in `behaviour/comments.md` (Comments, Editing, Deleting your own).
