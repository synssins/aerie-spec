# Q-0032: AI search limits, audit and access

Raised by: the spec room, while specifying the owner's request for AI search.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

How is AI search rate-limited per person, what does the audit log keep without storing page content, and who may call it?

## Answer

- Calls count against the existing per-person AI limit (`userPerMinute`) and the site's model-token limit (`tokenPerMinute`), shared with the other AI features, so an Admin has one place to set limits. Past the limit: `429` with `Retry-After`, and the model is not called.
- One audit entry per call that passes the availability check, including those refused by the rate limit: action `search.answer`, actor, time, `via`. The question, page texts, answer and page ids are not kept, in the audit log or the application log. The question is left out because people type personal and confidential details into questions.
- Available only with a session: guests get `401` and API tokens are refused, so the model cannot be driven by scripts. Off until an Admin sets the `search` feature to `Admins` or `Everyone`; when it is not available to the caller the call is `403`, and `MeResponse.aiSearch` tells the web app whether to show it.

For the owner: keeping the question text in the audit log would help investigate misuse but stores what people typed. This ruling does not keep it.

Written in `behaviour/ai-search.md` (When it is available, Limits, Audit); `search.answer` and `MeResponse.aiSearch` in `contracts/aerie/openapi.yaml`.
