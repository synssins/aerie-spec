# Q-0022: Are comments threaded or flat?

Raised by: the spec room, while specifying the owner's request for page comments.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

Should comments be flat, threaded to any depth, or something in between?

## Answer

One level of replies. A comment is either top-level or a reply to a top-level comment. A reply to a reply is `400`; clients reply to the top-level comment.

Reason: flat comments cannot hold a conversation, and deep threads are hard to read, moderate and page. One level covers asking and answering. It is also what the Data Center compatible comment model can carry when comments are exposed there (T2, question Q-0007), so no later migration is needed.

At most 100 replies per top-level comment; the 101st is `422`. This keeps every list answer bounded without paging replies.

Written in `behaviour/comments.md` (Replies).
