# Q-0031: What AI search answers, and when nothing is relevant

Raised by: the spec room, while specifying the owner's request for AI search.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

What does the answer look like, how are links made, and what does it say when nothing relevant is readable?

## Answer

- Three outcomes: `answered`, `no_answer` (pages found but they do not answer, or the model cited none of them), `nothing_found` (no usable page; the model is not called).
- Links are built only by aerie, from the pages it sent: the model cites `[1]`, `[2]`; aerie maps each to its page. Citation numbers that were not sent are removed. An answer that cites nothing counts as `no_answer`, because an answer with no source cannot be checked.
- For `no_answer` and `nothing_found`, the text is aerie's own fixed text, not the model's. It never says or hints whether unreadable pages or sensitive spaces exist; the `nothing_found` text is the same whether nothing matched anywhere or only pages the person cannot use matched.
- The answer is plain text; the web app shows no links, HTML or markup from the model, so a page that tries to steer the model cannot put a link in front of the reader.
- A model error or timeout is `503`.

Written in `behaviour/ai-search.md` (The answer); `SearchAnswerDto` in `contracts/aerie/openapi.yaml`.
