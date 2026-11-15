# Identity: build plan

Sources: L-0025, L-0100, L-0102, L-0105, L-0108, L-0109, L-0110, L-0111, L-0112, L-0113, L-0114, L-0115

How to build what [`identity.md`](identity.md) specifies, in slices the build room can take one at a time. Everything a builder needs is in `identity.md`, `common.md`, `contracts/datacenter/openapi.yaml` (the `oauth2` tag and the `bearer` and `oauth2` security schemes) and `contracts/aerie/openapi.yaml` (operations marked `x-aerie-status: planned`). No vendor material is needed.

Each slice lists what it must satisfy, its security requirements, what it reuses from aerie as the as-is contract shows it, and the lab `auth` suite cases that prove it. Case names are suggestions for the lab; the lab owns its suite.

## What already exists in aerie (as-is contract, L-0025)

| Existing piece | Where it shows | Reused for |
|---|---|---|
| API tokens `kbt_...`, kinds `Personal` and `Service`, per-space `Read`/`Write` scopes, expiry, revocation, `lastUsedAt`, admin list | `/api/v1/tokens`, `/api/v1/admin/tokens`, `ApiTokenDto`, `TokenScope`, `TokenSettingsDto` | Personal access tokens on `/rest/api` (slice 1, 2). The hash-and-lookup code is the model for OAuth token storage (slice 3) |
| Bearer scheme on `/api/v1` | `bearerAuth` security scheme, `x-kb-api-token` | The bearer parser (slice 1) |
| Local sign-in with lockout, sign-in codes (TOTP), the per-user code state, admin exemption | `/api/v1/auth/local/signin`, `/api/v1/auth/local/code`, `/api/v1/me/codes` (`MyCodes`), `/api/v1/admin/settings/codes`, `/api/v1/admin/users/{id}/codes/exempt` | Password check and lockout for Basic (slice 1); the sign-in step inside the OAuth flow (slice 5) |
| Session cookie `__Host-kb` | `cookieAuth` security scheme | Cookie callers on `/rest/api` (slice 1); the consent page's API calls (slice 5) |
| Administrator flag, space roles (`Reader`, `Contributor`, `Supervisor`, `Owner`), groups, custom roles | `MeResponse.isAdmin`, `SpaceRole`, `/api/v1/admin/roles` | The permission check after the scope check (slice 2) |
| Audit log with actor, action, target, `via`, `token` | `/api/v1/admin/audit`, `AuditEntryDto` | Every audit event in `identity.md` (all slices) |
| Problem bodies with `429` for too many calls | `Problem`, the `default` responses | Rate-limit answers on aerie's own API (slice 4, 5) |

## Order

1. Credential resolution on the compatible surface (Basic, personal access tokens, cookie)
2. Scope enforcement
3. OAuth data model
4. Client registration (API, admin page, settings)
5. Authorize endpoint and consent
6. Token endpoint: code exchange
7. OAuth access tokens on `/rest/api`
8. Refresh, rotation and reuse detection
9. Revocation, connected applications, metadata
10. Hardening pass: rate limits and audit review

Slices 1 and 2 do not depend on OAuth and unblock the lab's Basic and token cases at once. Slices 3 to 9 go in order. Slice 10 is a check across all of them.

## Slice 1: credential resolution

**Build.** One piece of middleware in front of every `/rest/api` route that turns the request into a caller: a user (with how they authenticated), a service account, or anonymous. Order and rules from `identity.md` "The callers aerie must tell apart":

- `Authorization: Bearer kbt_...`: look up the hash; check expiry, revocation, the owner (or service account) is enabled; update `lastUsedAt` (batched is fine).
- `Authorization: Basic`: decode; if the secret starts with `kbt_` (later also `kbo_`), handle it as that token and require the username to match the owner, case-insensitive. Otherwise check the password through the same code and counters as `/api/v1/auth/local/signin`.
- Password Basic for an account with sign-in codes on (user-chosen or required by policy, and not exempted): `401`, same body and timing as a wrong password; audit the real reason.
- Password Basic when the site setting turns it off: `401`, audited.
- Cookie: the existing session. Non-`GET`/`HEAD`/`OPTIONS` requires `X-Atlassian-Token: no-check` or `nocheck`, else `403`.
- No credentials: anonymous.
- An `Authorization` header that is present but unusable: `401`, never anonymous.

**Security requirements.**

- No session is created and no cookie is set for Basic or bearer requests.
- Password checks take the same time whether the user exists, the password is wrong, or codes are on (compare against a dummy hash when the user is unknown).
- Failures count against the sign-in lockout per account and per source address; past the limit, `429` with `Retry-After`, without checking the password.
- No `WWW-Authenticate: Basic` on `401` from `/rest/api`.
- Tokens are never logged, not even on failure; log the prefix and token id only.

**Lab cases.**

- `auth.basic.password.ok`: a user without codes reads `/rest/api/user/current` with Basic.
- `auth.basic.password.wrong`: `401`, `RestError` body.
- `auth.basic.codes_on.refused`: an administrator with codes gets `401` with the right password; the body equals the wrong-password body.
- `auth.basic.codes_exempt.ok`: the same administrator, exempted, gets `200`.
- `auth.basic.token_in_password.ok`: Basic with username and a `kbt_` token gets `200`, also for an account with codes on.
- `auth.basic.token_in_password.wrong_user`: `401`.
- `auth.basic.lockout`: repeated failures end in `429` with `Retry-After`.
- `auth.pat.bearer.ok`, `auth.pat.bearer.revoked`, `auth.pat.bearer.expired`: `200`, `401`, `401`.
- `auth.no_cookie_set`: no `Set-Cookie` on Basic or bearer responses.
- `auth.cookie.write_needs_xsrf`: a cookie `POST` without the header gets `403`.
- `auth.header.malformed`: `Authorization: Bearer` with no token, or an unknown scheme, gets `401`.

## Slice 2: scope enforcement

**Build.** A check after credential resolution and before the operation's own permission check:

- Each compatible operation's required scope is derived: `GET`/`HEAD` need `READ`; other methods need `WRITE`; an operation carrying `x-aerie-scope: ADMIN` in the contract needs `ADMIN`. No T1 operation carries it. Generate the table from the contract at build time rather than writing it by hand.
- For a `kbt_` token: `Read` tokens pass only `READ`; space-limited tokens make other spaces invisible (`404` on reads, left out of lists, as `common.md` says). Passwords and cookies pass every scope.
- For an OAuth token (wired in slice 7): its granted scope set, expanded by the implications table in `identity.md`.
- On failure for a bearer token: `403` with `WWW-Authenticate: Bearer error="insufficient_scope", scope="..."` and a `RestError` body.
- After the scope check, the normal permission check as the user. Scope can never add access.

**Security requirements.** Default deny: a compatible route with no derivable scope fails the build (a test asserts every route has one). Space limits apply to search and lists, not only to direct reads.

**Lab cases.**

- `auth.pat.read_only.write_refused`: a `Read` token gets `403` on `POST /rest/api/content`.
- `auth.pat.space_limited.hidden`: a token limited to space A gets `404` for a page in space B and does not see it in `/rest/api/search`.
- `auth.scope.permission_still_applies`: a `Write` token of a `Reader` cannot edit a page (`403`).

## Slice 3: OAuth data model

**Build.** Storage for, at least:

| Record | Fields that matter |
|---|---|
| Client | id, `clientId` (unique), secret hash (nullable for public), name, type, redirect URIs, allowed scopes, `requirePkce`, enabled, created by and at, secret rotated at |
| Pending authorization request | id, client, user (once signed in), redirect URI, scopes asked, `state`, PKCE challenge and method, created at, expires at (10 minutes), decided |
| Grant (consent) | id, user, client, granted scopes, created at, last used at. One per user and client |
| Authorization code | hash, grant, redirect URI, scopes, PKCE challenge, expires at, used at |
| Token family | id, grant, created at, revoked at, revoke reason |
| Access token | hash, prefix, family, scopes, expires at, revoked at |
| Refresh token | hash, prefix, family, expires at (idle), rotated at (set when replaced), revoked at |
| Settings | the `CompatAuthSettingsDto` fields |

**Security requirements.**

- Secrets, codes and tokens are stored only as hashes: SHA-256 for tokens and codes (they are high-entropy), the same password hash aerie uses for passwords for client secrets.
- Generated values use a cryptographic random source: client secrets 64 characters, tokens and codes at least 256 bits.
- Lookups by hash use an index; comparisons of secrets are constant-time.
- Expired records are pruned on a schedule (every few minutes is enough); pruning never deletes a used code or rotated refresh token before its family expires, because reuse detection needs them.
- Disabling a user, disabling or deleting a client, and revoking a grant each make every dependent token fail at once (check status at validation time, or cascade revocation; either is fine as long as the effect is immediate).

**Lab cases.** None directly; slices 4 to 9 cover it.

## Slice 4: client registration

**Build.**

- The `oauth-clients` operations and `auth.compat.read` / `auth.compat.save` in `contracts/aerie/openapi.yaml`.
- An "Applications" admin page: list, create (name, type, redirect URIs, scopes), show the secret once with a copy button and a clear "you will not see this again", rotate secret, edit, disable, delete with confirmation.
- The compatible-surface sign-in settings on the same page or next to code settings.

**Rules to enforce.** From `identity.md` "Clients", "Who registers clients" and "Redirect URI rules":

- Administrators only; everyone else `403`.
- Redirect URIs: absolute, no fragment, `https` except loopback `http`, private-use schemes with a dot, or the test-install setting. At least one per client.
- Scopes: any of `READ`, `WRITE`, `ADMIN`, `SYSTEM_ADMIN`; at least one. Others are `400`.
- `requirePkce` can be `false` only for `confidential` clients.
- Settings outside the ranges in `identity.md` "Tokens" are `400`.

**Security requirements.** Secret appears in exactly two responses (create, rotate) and nowhere else, including logs and audit. All changes audited (`oauth.client.*`). Deleting a client cascades to grants and tokens.

**Lab cases.**

- `auth.client.register`: an administrator creates a confidential client through `/api/v1/admin/oauth/clients` and gets a `clientId` and secret.
- `auth.client.register.non_admin`: `403`.
- `auth.client.redirect.rejects`: `http://example.com/cb`, a URI with `#frag`, and `*` patterns are each `400`.
- `auth.client.secret_not_readable`: list and read never show a secret.

## Slice 5: authorize endpoint and consent

**Build.**

- `GET /rest/oauth2/latest/authorize`, exactly as the contract and `identity.md` "The authorization request" say.
- The sign-in hand-off: an unsigned user goes through aerie's normal sign-in (password, then code, or single sign-on) and returns to the pending request.
- A consent page in aerie's web app, at a route of aerie's choosing, using `oauth.request.read` and `oauth.request.decide`.
- Skip the page when the user's existing grant already covers the scopes asked for.

**Security requirements.**

- Validate `client_id` and `redirect_uri` first. If either is bad, show an error page and never redirect.
- Exact redirect matching (`identity.md`); no normalisation beyond lowercasing scheme and host; loopback port ignored.
- PKCE: required for public clients and for confidential clients with `requirePkce`; only `S256` (and its spelling `sha256`).
- The consent page is protected against framing (`frame-ancestors 'none'`) so it cannot be clickjacked.
- `oauth.request.decide` needs the session cookie, checks that the pending request belongs to the signed-in user and is undecided and unexpired, and is protected by aerie's existing CSRF defence for cookie calls.
- `state` is returned byte for byte.
- The redirect carries `code` and `state` only in the query; nothing in a fragment.
- Rate limit authorize requests per source address (aerie's existing limiter; `429` page past it).
- Audit `oauth.consent.grant` and `oauth.consent.deny`.

**Lab cases.**

- `auth.oauth.authorize.bad_redirect`: no redirect, error page.
- `auth.oauth.authorize.unknown_client`: no redirect, error page.
- `auth.oauth.authorize.missing_pkce`: redirect with `error=invalid_request`.
- `auth.oauth.authorize.plain_pkce`: redirect with `error=invalid_request`.
- `auth.oauth.authorize.bad_scope`: redirect with `error=invalid_scope`.
- `auth.oauth.consent.deny`: redirect with `error=access_denied` and the same `state`.
- `auth.oauth.consent.remembered`: a second authorization for the same scopes skips the page.
- `auth.oauth.signin_with_codes`: an administrator with codes signs in with password and code, then consents.

## Slice 6: token endpoint, code exchange

**Build.** `POST /rest/oauth2/latest/token` with `grant_type=authorization_code`, as `identity.md` "The token endpoint" says: parameters from body or query (body wins), client authentication by parameters or Basic, the JSON response with `created_at` and `scope`.

**Security requirements.**

- Codes are single use. A second use is `invalid_grant` and revokes the family issued from the first use; audit `oauth.token.reuse`.
- Code lifetime from settings (default 300 s).
- `redirect_uri` must equal the one in the authorize request.
- PKCE: `BASE64URL(SHA256(code_verifier))` must equal the stored challenge; compare in constant time.
- The code must belong to the authenticating client.
- `Cache-Control: no-store` and `Pragma: no-cache` on every response.
- Rate limit per client and per source address; past it, `429` with `Retry-After` and the body `{"error":"temporarily_unavailable"}` (aerie's choice; RFC 6749 has no token-endpoint code for this).
- Audit `oauth.token.issue`.

**Lab cases.**

- `auth.oauth.pkce.full_flow`: a public client with PKCE gets tokens and calls `/rest/api/user/current`.
- `auth.oauth.confidential.full_flow`: a confidential client, secret in the query string as the vendor's examples do, then again with HTTP Basic.
- `auth.oauth.code.reuse`: second exchange gets `invalid_grant`, and the access token from the first exchange stops working.
- `auth.oauth.code.wrong_verifier`: `invalid_grant`.
- `auth.oauth.code.wrong_redirect`: `invalid_grant`.
- `auth.oauth.client.bad_secret`: `401` `invalid_client`.

## Slice 7: OAuth access tokens on `/rest/api`

**Build.** Extend slice 1 to recognise `kbo_` tokens (bearer, and in the Basic password field) and slice 2 to apply their scopes.

**Security requirements.**

- Reject expired, revoked, disabled-client and disabled-user tokens with `401` and `WWW-Authenticate: Bearer error="invalid_token"`.
- `kbo_` tokens on aerie's own `/api/v1` get `401`.
- Tokens only in the `Authorization` header; a token in the query string is ignored.

**Lab cases.**

- `auth.oauth.read_scope.write_refused`: a `READ` token gets `403` with `insufficient_scope` on a write.
- `auth.oauth.write_scope.ok`: a `WRITE` token creates a page in a space where the user is a `Contributor`.
- `auth.oauth.admin_scope.non_admin_user`: an `ADMIN`-scoped token of a non-administrator gains nothing beyond `WRITE`.
- `auth.oauth.token_not_on_aerie_api`: a `kbo_` token on `/api/v1/me` gets `401`.
- `auth.oauth.token_in_basic`: a `kbo_` token in the Basic password field works.

## Slice 8: refresh, rotation, reuse detection

**Build.** `grant_type=refresh_token` on the token endpoint, as `identity.md` "Refresh rotation and reuse detection" says.

**Security requirements.**

- Every refresh invalidates the presented refresh token and its access token, and issues a new pair in the same family.
- A rotated refresh token presented again revokes the whole family (`invalid_grant`), keeps the consent, and is audited as `oauth.token.reuse`.
- The rotation is atomic: two concurrent refreshes with one token cannot both succeed (a conditional update on "not yet rotated").
- Idle refresh lifetime and the optional absolute grant lifetime from settings.
- `scope` on refresh may only narrow.
- At most 25 active grants per user and client; the 26th revokes the least recently used.

**Lab cases.**

- `auth.oauth.refresh.ok`: new pair; the old access token now gets `401`.
- `auth.oauth.refresh.after_access_expiry`: refresh still works after the access token expired (lab sets a short access lifetime through settings).
- `auth.oauth.refresh.reuse`: presenting the old refresh token again gets `invalid_grant`, and the newest access token stops working too.
- `auth.oauth.refresh.narrow_scope`: `scope=READ` on a `READ WRITE` grant gives a `READ` token; asking for more gets `invalid_scope`.

## Slice 9: revocation, connected applications, metadata

**Build.**

- `POST /rest/oauth2/latest/revoke` (RFC 7009).
- `oauth.grant.list` and `oauth.grant.revoke`, and a "Connected applications" page in the user's profile.
- `GET /.well-known/oauth-authorization-server`.

**Security requirements.** Revoke always answers `200` for a token that is unknown, already revoked or another client's (and does nothing in the last case). Revoking a refresh token ends its family. A user can see and revoke only their own grants. Audit `oauth.grant.revoke`.

**Lab cases.**

- `auth.oauth.revoke.refresh`: after revoking, both refresh and access tokens fail.
- `auth.oauth.revoke.unknown`: `200`.
- `auth.oauth.user_revokes_grant`: revoking in the profile ends the client's tokens at once.
- `auth.oauth.metadata`: the document lists the endpoints and `S256`.

## Slice 10: hardening pass

**Check, across slices 1 to 9.**

- Rate limits exist and answer correctly on: Basic failures (`429` with `Retry-After`), authorize, token, revoke.
- Every audit action in `identity.md` "Audit" is written, with actor, client id, source address and outcome, and no secret, code or token anywhere in the audit log or application logs (grep the lab run's logs for `kbt_`, `kbo_`, `kbr_` followed by more than the prefix).
- HTTPS: in a production install, the OAuth endpoints refuse to work when the site base URL is plain `http` (the vendor has the same rule, L-0100), unless the test-install setting is on.
- `Cache-Control: no-store` on token and revoke responses.
- A dependency check of the cryptographic primitives used (random source, SHA-256, constant-time compare).

**Lab cases.** `auth.audit.no_secrets` (log scan), `auth.ratelimit.token`, `auth.ratelimit.basic`.

## Decisions for the owner

Real product choices only. Each has a default that the build follows unless the owner says otherwise.

1. **Password Basic for accounts with sign-in codes.** Default: refused; those accounts use a personal access token (also in the Basic password field). Alternative: allow it, which lets scripts skip the second factor.
2. **PKCE for server-side (confidential) clients.** Default: required, with a per-client switch an administrator can turn off for an older integration that cannot send it. Alternative: never required for them, which is what the vendor allows but current OAuth security advice does not.
3. **Client secret visibility.** Default: shown once, can be rotated, never viewed again. The vendor lets administrators view it later. Alternative: store it recoverably to match.
4. **Service accounts (OAuth client credentials) and the special scopes `READ_ALL`, `MANAGE_SUBSCRIPTIONS`, `JSM_KB`.** Default: not in the first build; aerie's existing `Service` tokens cover scripted access. Alternative: add them now, which means machine identities that bypass space permissions.
5. **The vendor's token-management API** (creating personal access tokens through a vendor path). Default: later; tokens are made in aerie's UI or its own API. Alternative: add it to the contract now.
6. **OAuth tokens on aerie's own API.** Default: only on the compatible surfaces. Alternative: also accept them on `/api/v1`, which would let third-party apps reach aerie's own features.
7. **Consent can never be skipped.** Default: every user approves every client once. Alternative: let administrators pre-approve trusted internal clients.
