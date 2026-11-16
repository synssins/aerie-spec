# Q-0040: Which AI limits apply to AI search

Raised by: the build room, against `contracts-v8`.
Status: answered in pull request `ai-search-limits`.

## Where

Aerie's own API under `/api/v1`: the planned `search.answer` call and the existing AI settings (`userPerMinute`, `tokenPerMinute`).

## Question

`behaviour/ai-search.md` (Limits) and question Q-0032 say AI search counts against the site's model-token limit, `tokenPerMinute`. In aerie, `tokenPerMinute` is the number of AI calls a minute allowed per API token, not a limit on model tokens, and there is no site-wide model-token limit. Which limits apply to AI search?

## Answer

- Only the per-person limit, `userPerMinute`, through the same limiter as aerie's other AI features. Past it: `429` with `Retry-After`, and the model is not called.
- The per-API-token limit, `tokenPerMinute`, does not apply. AI search is session only and refuses API tokens (`403` for a valid token, question Q-0039), so no call can reach that limit.
- No new setting. Aerie has no site-wide model-token limit, and AI search does not add one.

The Limits text in `behaviour/ai-search.md` and the limits line in question Q-0032 are corrected. No contract change: the `search.answer` description already names only the per-person limit.

Written in `behaviour/ai-search.md` (Limits); question Q-0032.
