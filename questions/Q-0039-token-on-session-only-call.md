# Q-0039: Which status an API token gets on a session-only call

Raised by: the build room, against `contracts-v7`.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: every call marked `x-kb-api-token: closed`, including the planned comment, rating and AI search calls.

## Question

The `contracts-v7` lab cases `comments.token.read_only`, `ratings.token.refused` and `ai.search.token.refused` expect `401` for an API token on a call closed to tokens. Aerie today answers such a call with `403` when the token is valid, and `401` only when the token is unknown or expired; `behaviour/identity.md` makes the same split. Which is right?

## Answer

Aerie's rule stands:

- A valid token on a call closed to tokens: `403` with a problem body. The token proved who the caller is; the call is simply not open to tokens.
- An unknown, expired or revoked token: `401`.

The three lab cases and the matching text in `comments.md`, `ratings.md` and `ai-search.md` now say `403` for a valid token and `401` for a bad one. No contract response changes: every closed call already lists `401` ("Not signed in, or a bad token") and `403` ("Not allowed").

Written in `behaviour/comments.md`, `behaviour/ratings.md`, `behaviour/ai-search.md` (rules and lab cases); question Q-0032.
