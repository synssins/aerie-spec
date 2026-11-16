# Q-0030: What the model may see in AI search

Raised by: the spec room, while specifying the owner's request for AI search.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

AI search answers only from pages the person may read, and sensitive spaces never reach the model. What exactly is sent, and how is it bounded?

## Answer

- Aerie runs its existing search for the question as the asking person, drops hits in sensitive spaces (even ones the person can read), drops everything that is not a page, re-checks read access on each page, and keeps the first 8.
- Each page's current text, title and space name are sent, cut so that the question and the texts fit the existing `maxInputChars` AI setting, with an equal share per page.
- Not sent: comments, ratings, attachments, history, and anything about dropped pages.
- The model gets no tools and cannot fetch or search. Nothing leaves aerie except the request to the configured model endpoint. The texts and answer are not stored, cached or logged. (The question was also not stored; that part is superseded: the question is now kept in the audit entry, question Q-0032.)
- Page text is passed as fenced, numbered data after aerie's fixed instructions; it is never treated as instructions.

Eight pages keeps the request within typical model limits and the answer focused; it is a constant in this first version, not a setting.

Written in `behaviour/ai-search.md` (What the model may see, The guardrails).
