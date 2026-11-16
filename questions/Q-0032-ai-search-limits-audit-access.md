# Q-0032: AI search limits, audit and access

Raised by: the spec room, while specifying the owner's request for AI search.
Status: answered in pull request `phase1-features-1`; the audit part revised by the owner in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

How is AI search rate-limited per person, what does the audit log keep without storing page content, and who may call it?

## Revised answer (owner, `phase1-features-2`)

The owner reversed the audit default: question history matters for investigating prompt injection attempts. Limits and access are unchanged.

- Every AI search by a signed-in person is recorded, whatever the outcome, in the audit entry itself (`AuditEntryDto.aiSearch`), not in a linked record. That puts it under the audit chain, the audit log's readers, its retention and its shipping to log targets (SIEM) with no new machinery.
- Recorded: who asked, when, the question exactly as sent (cut at 2000 characters, with its full length), the outcome (`answered`, `no_answer`, `nothing_found`, `refused`, `rate_limited`, or `failed` for a model error or timeout), the space filter, and the id and version of each page given to the model.
- Not recorded: page text and the answer text. Page ids with versions rebuild exactly what the model saw from page history; the answer is drawn from page content, and a second copy of that content in the audit log adds no evidence.
- Readable by Admins only (the audit log's readers). Kept as long as the audit log keeps entries. Never written to the application log.
- The ask box tells people, before they type, that questions are recorded with their name and can be read by administrators.

## Earlier answer (superseded for the audit part)

- Calls count against the existing per-person AI limit (`userPerMinute`) and the site's model-token limit (`tokenPerMinute`), shared with the other AI features, so an Admin has one place to set limits. Past the limit: `429` with `Retry-After`, and the model is not called.
- One audit entry per call that passes the availability check, including those refused by the rate limit: action `search.answer`, actor, time, `via`. The question, page texts, answer and page ids are not kept, in the audit log or the application log. The question is left out because people type personal and confidential details into questions.
- Available only with a session: guests get `401` and API tokens are refused, so the model cannot be driven by scripts. Off until an Admin sets the `search` feature to `Admins` or `Everyone`; when it is not available to the caller the call is `403`, and `MeResponse.aiSearch` tells the web app whether to show it.

For the owner: keeping the question text in the audit log would help investigate misuse but stores what people typed. This ruling does not keep it.

Written in `behaviour/ai-search.md` (When it is available, Limits, Audit); `search.answer` and `MeResponse.aiSearch` in `contracts/aerie/openapi.yaml`.
