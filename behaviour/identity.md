# Identity: authentication and authorization on the compatible surfaces

Sources: L-0007, L-0010, L-0025, L-0100, L-0101, L-0102, L-0103, L-0104, L-0105, L-0106, L-0107, L-0108, L-0109, L-0110, L-0111, L-0112, L-0113, L-0114, L-0115

This file says how a caller proves who it is on the Data Center compatible surface (`/rest/api`, `/rest/oauth2`) and what it may then do. It covers HTTP Basic, personal access tokens, OAuth 2.0 (authorization code with PKCE, refresh, revocation, scopes, client registration, consent) and how all of these sit next to aerie's own sign-in, sessions and API tokens. The build order is in [`identity-plan.md`](identity-plan.md).

OIDC sign-in and LDAP directories (listed for this file in `README.md`) are aerie's own sign-in features, not part of any vendor API a client calls. They are left for a later file; nothing here depends on them.

**This file supersedes the provisional ruling** "Accounts with sign-in codes (provisional)" in `common.md` and question Q-0005. The ruling stands as written; this file adds the token routes such accounts use instead (see [Basic auth](#http-basic)). When both branches are merged, the provisional section in `common.md` should be reduced to a pointer here.

Where a rule is aerie's own choice rather than a published vendor fact, it says so.

## Which surface is which

| Part | Data Center | Cloud |
|---|---|---|
| HTTP Basic | Username and password, or username and a token (below) | Accepted for direct API calls (L-0010). The secret is not the account password. Specified with the Cloud contract |
| Personal access tokens | Bearer tokens bound to a user (L-0102) | None |
| OAuth 2.0 | Aerie is the authorization server, under `/rest/oauth2/latest/` (L-0100) | The vendor runs a central authorization server; apps also use JWT (L-0010). Different scope strings. Specified with the Cloud contract |
| Client credentials (service accounts) | Exists in recent Data Center releases (L-0106, L-0107). Not in aerie's first build; see the decisions in `identity-plan.md` | Not covered |

Everything below is Data Center unless marked otherwise. Cloud comes after Data Center; only its differences are noted.

## The callers aerie must tell apart

Every request on the compatible surface resolves to exactly one of these, checked in this order. The first credential present decides; aerie never combines two.

1. `Authorization: Bearer <token>`: a personal access token or an OAuth 2.0 access token.
2. `Authorization: Basic <base64(username:secret)>`: a password, or a token in the password field.
3. Aerie's own session cookie (`__Host-kb`, from the as-is contract): a user signed in to the web app.
4. Nothing: the anonymous user.

A request that sends an `Authorization` header aerie cannot use (an unknown scheme, a malformed value) gets `401`. It is never treated as anonymous.

## Personal access tokens

Data Center personal access tokens are long random secrets bound to one user. They act with that user's full permissions and are sent as `Authorization: Bearer <token>` (L-0102).

Aerie already has these. Its API tokens (`kbt_...`, from `/api/v1/tokens` in the as-is contract) are the personal access tokens of the compatible surface. No second token type is added.

- A `kbt_` token of kind `Personal` acts as its owner. One of kind `Service` acts as the service account it was made for (`accountId`).
- Aerie's tokens carry their own scopes (`TokenScope`: a space or all spaces, `Read` or `Write`). They keep applying on the compatible surface, on top of the user's permissions:
  - A `Read` token may call only `GET` and `HEAD`. Any other method gets `403` and nothing changes.
  - A token limited to some spaces sees only those spaces. Content in other spaces behaves as if it does not exist (`404` on reads, left out of lists), the same as the hiding rule in `common.md`.
  - Vendor tokens have no scopes, so a vendor client is unaware of these limits. A token with all spaces and `Write` behaves like a vendor token.
- An expired or revoked token gets `401`.
- Using a token updates its `lastUsedAt` (as aerie does today). Aerie may batch these updates; the value may be up to five minutes behind (aerie's choice).
- Token-authenticated requests never create a session and never set a cookie (L-0102 describes the same for the vendor).
- Creating tokens through the vendor token API (`/rest/pat/latest/tokens`, L-0102) is not in this contract. Tokens are made in aerie's UI or through `/api/v1/tokens`. Adding the vendor token-management calls is a follow-up.

## HTTP Basic

`Authorization: Basic base64(username:secret)` is accepted on every compatible operation (`common.md`). The secret is one of:

- **The account password.** Checked like aerie's own sign-in (`/api/v1/auth/local/signin`), with the same lockout counters, but no session is created and no cookie is set.
- **A token.** A personal access token or an OAuth 2.0 access token in the password field. The vendor's OAuth provider accepts its access tokens this way by default (L-0105), and some clients only know Basic. Aerie recognises a token by its prefix (`kbt_` or `kbo_`, below) and then treats the request exactly like the bearer form. The username must be the token owner's username, compared without regard to case; a mismatch gets `401` (aerie's choice: a wrong username is a sign of a mix-up, not something to ignore).

### Accounts with sign-in codes

Aerie requires administrators to use time-based sign-in codes, and other users may turn them on (`/api/v1/admin/settings/codes`, `/api/v1/me/codes` in the as-is contract). Basic has no place for a code.

- **Password in Basic, codes on:** refused with `401`. The response is identical to a wrong password, whether or not the password was right, so Basic cannot be used to test a password while skipping the code. The real reason goes to the audit log. This covers codes the user turned on and codes a policy requires. (Same as the provisional ruling in `common.md`.)
- **Token in Basic, codes on:** accepted. A token was created inside a session that already passed the code, so it is the intended route for such accounts.
- **Bearer tokens and OAuth:** unaffected by codes. The OAuth sign-in step (below) runs aerie's normal sign-in, including the code.
- Accounts without codes, including those an administrator exempted (`/api/v1/admin/users/{id}/codes/exempt`), may use the password in Basic.
- Administrators can turn off password Basic for everyone with a site setting (aerie's choice, default on). Token Basic is not affected.

### Failed Basic attempts

- A wrong password or unknown username gets `401` with the `RestError` body from `common.md`.
- Failures count against the same per-account and per-address limits as aerie's sign-in page. Past the limit, aerie answers `429` with `Retry-After` and does not check the password at all until the window passes (aerie's choice).
- A `401` on the compatible surface carries no `WWW-Authenticate: Basic` challenge, so a browser on aerie's own pages never shows a password dialog (aerie's choice; the lab should confirm vendor clients do not need the challenge). A failed bearer token is different; see [Errors on protected calls](#errors-on-protected-calls).

## Sessions and aerie's own API

- Aerie's session cookie is accepted on the compatible surface, so the web app (and a signed-in browser) can call it. For any method other than `GET`, `HEAD` and `OPTIONS` made with the cookie, the `X-Atlassian-Token: no-check` header (or `nocheck`) is required, whatever the content type; without it the response is `403` (aerie's choice, stricter than the form-only rule in `common.md`, because a cookie is sent by the browser on its own).
- Bearer and Basic callers are not subject to that header rule beyond what `common.md` already says.
- OAuth access tokens (`kbo_`) work only on the compatible surfaces (`/rest/api` now, Cloud paths later). On aerie's own `/api/v1` they get `401`. Aerie's own API keeps its two schemes, the cookie and `kbt_` tokens on the routes marked for tokens, unchanged.
- Signing out of aerie (`/api/v1/auth/signout`) ends the session only. Tokens and OAuth grants stay valid until they expire or are revoked.
- When an administrator disables a user, every token and OAuth grant of that user stops working at once.

## OAuth 2.0 (Data Center)

Aerie acts as an OAuth 2.0 authorization server for clients that act on a user's behalf. The flow is the authorization code grant (RFC 6749 §4.1, L-0108) with PKCE (RFC 7636, L-0110). The implicit grant and the password grant are not offered (L-0100; RFC 9700 §2.1.2 and §2.4, L-0113).

### Endpoints

All under the site base URL with any context prefix (`common.md`).

| Endpoint | Method | Purpose | Origin |
|---|---|---|---|
| `/rest/oauth2/latest/authorize` | `GET` | Start the flow in the user's browser: sign-in, consent, redirect back with a code | Vendor (L-0100) |
| `/rest/oauth2/latest/token` | `POST` | Exchange a code, or a refresh token, for tokens | Vendor (L-0100) |
| `/rest/oauth2/latest/revoke` | `POST` | Revoke a refresh or access token (RFC 7009) | Aerie's own addition |
| `/.well-known/oauth-authorization-server` | `GET` | Server metadata (RFC 8414) | Aerie's own addition |

Vendor clients only know the first two. The other two are standard extras that do no harm to them.

### Clients

A client is an application an administrator registered. It has:

| Field | Meaning |
|---|---|
| `clientId` | Public identifier, generated by aerie, 32 random characters from `[A-Za-z0-9]` |
| `clientSecret` | Generated by aerie, 64 random characters, shown once when created or rotated. Stored only as a hash. Absent for public clients |
| `name` | Shown to users on the consent screen |
| `type` | `confidential` (has a secret) or `public` (no secret; a desktop, mobile or browser app that cannot keep one) |
| `redirectUris` | One or more exact redirect URIs |
| `scopes` | The scopes this client may ever ask for (see [Scopes](#scopes)) |
| `requirePkce` | `true` by default. May be turned off only for `confidential` clients |
| `enabled` | A disabled client cannot start flows, exchange codes or refresh; its access tokens stop working |
| `createdBy`, `createdAt`, `secretRotatedAt` | Bookkeeping |

The vendor issues a client id and secret for every incoming link and lets the administrator view the credentials again later (L-0101). Aerie differs on one point: the secret is shown once and can only be rotated, never viewed again (aerie's choice; storing it recoverably would defeat hashing).

### Who registers clients, and where

- Only administrators (`isAdmin` in aerie's `MeResponse`). There is no self-service registration and no dynamic registration (RFC 7591).
- In the web app: an "Applications" page under administration, next to API tokens. It lists clients, creates one (name, type, redirect URIs, scopes), shows the secret once, rotates the secret, edits, disables and deletes.
- Through aerie's own API: the new `/api/v1/admin/oauth/clients` operations in `contracts/aerie/openapi.yaml` (marked not yet built). They are cookie-only like the other admin routes, so the lab signs in as an administrator to seed a client. This closes the lab gap "no API call to register an OAuth client".
- Deleting a client revokes every grant and token issued to it. Rotating the secret does not revoke existing tokens; it only changes what the client must send to the token endpoint.

### Redirect URI rules

From RFC 9700 §2.1 and §4.1 (L-0113), RFC 6749 §3.1.2 (L-0108), and the vendor's HTTPS rule (L-0100, L-0105):

- Matching is exact, character for character, after nothing more than the scheme and host are lowercased. No wildcards, no prefix matching, no ignoring of query strings or trailing slashes.
- A registered URI must be absolute and must not contain a fragment (`#`).
- `https` is required, except:
  - `http` on a loopback host (`127.0.0.1`, `[::1]`, `localhost`), for native apps. For loopback URIs the port is ignored when matching, as RFC 8252 §7.3 asks; everything else must still match.
  - Private-use schemes for native apps (for example `com.example.app:/callback`), which must contain a dot in the scheme.
  - A site setting "allow plain http redirect URIs" for test installs, default off, mirroring the vendor's system property (L-0105).
- `redirect_uri` is required on every authorize request, even if the client has only one registered URI (the vendor marks it required, L-0100).
- If `client_id` is unknown or disabled, or `redirect_uri` does not match, aerie shows an error page to the user and **never redirects** (RFC 6749 §4.1.2.1). Every other error is sent back to the redirect URI.

### The authorization request

`GET /rest/oauth2/latest/authorize` with query parameters (names from L-0100; meaning from RFC 6749 and 7636):

| Parameter | Required | Rule |
|---|---|---|
| `response_type` | Yes | Must be `code`. Anything else: `unsupported_response_type` |
| `client_id` | Yes | A registered, enabled client |
| `redirect_uri` | Yes | Exact match, see above |
| `scope` | Yes | One or more scope names, separated by spaces. Missing or not allowed for the client: `invalid_scope` |
| `state` | No | Returned unchanged. Aerie does not require it but recommends it |
| `code_challenge` | When PKCE applies | 43 to 128 characters from `[A-Za-z0-9-._~]` |
| `code_challenge_method` | With `code_challenge` | `S256`. The spelling `sha256` used in the vendor's table (L-0100) is accepted as the same thing. `plain` is refused with `invalid_request` (RFC 9700 §2.1.1; a decision for the owner, see the plan) |

PKCE applies when the client is public, or is confidential with `requirePkce` on (the default). If it applies and `code_challenge` is missing, the answer is `invalid_request`.

What the user sees:

1. Not signed in to aerie: aerie's normal sign-in page, including the sign-in code step and any single sign-on aerie offers. After sign-in the user comes back to the same authorize request.
2. Consent (below).
3. The browser is redirected to `redirect_uri` with `code` and `state` (if sent). On refusal or error, with `error`, `error_description` and `state` instead.

The vendor's consent page lives at its own servlet path (L-0100). Clients never call it directly; they only follow the redirect. Aerie shows consent at a route of its own web app, so that path is not part of the contract.

### Consent

- The consent screen names the client, the scopes asked for in plain words, and the account that will be used, with "Allow" and "Deny".
- Consent is remembered per user and client as a grant. A later request for the same or fewer scopes skips the screen. A request for more scopes shows it again for the new total.
- Aerie never skips consent for any client (aerie's choice; administrators cannot pre-approve clients).
- Users see their grants on a "Connected applications" page in their profile and can revoke any of them (the vendor offers the same, L-0104). Revoking a grant ends all its tokens at once.
- The consent decision is posted by aerie's web app through `/api/v1/oauth/authorize-requests/{id}/decision` (new in `contracts/aerie/openapi.yaml`, not yet built). That call needs the session cookie and checks that the pending request belongs to the signed-in user, so a third-party page cannot approve on the user's behalf.
- Deny redirects with `error=access_denied`.

### Authorization codes

- Random, at least 256 bits, stored only as a hash.
- Valid for 300 seconds (aerie's choice; the vendor allows up to 600, L-0105, and RFC 6749 §4.1.2 says at most 10 minutes).
- Single use. A second exchange of the same code fails with `invalid_grant` and revokes every token already issued from that code (RFC 6749 §4.1.2).
- Bound to the client, the redirect URI, the user, the granted scopes and the PKCE challenge.

### The token endpoint

`POST /rest/oauth2/latest/token`.

Parameters arrive as an `application/x-www-form-urlencoded` body (RFC 6749 §4.1.3). The vendor's own examples put them in the query string of the POST instead (L-0100), so aerie also reads them from the query string. If a parameter is in both, the body wins. Either way, the response sets `Cache-Control: no-store` and `Pragma: no-cache`.

The XSRF header rule in `common.md` does not apply to this endpoint (it is called by servers, not browsers).

Client authentication, for confidential clients:

- `client_id` and `client_secret` as parameters (what the vendor's examples do, L-0100), or
- HTTP Basic with the client id and secret (RFC 6749 §2.3.1).
- Using both, with different values, is `invalid_request`.
- Public clients send `client_id` only.

**Exchanging a code** (`grant_type=authorization_code`): `code`, `redirect_uri` (must equal the one in the authorize request), `code_verifier` when PKCE applied (43 to 128 characters from `[A-Za-z0-9-._~]`), plus client authentication.

**Refreshing** (`grant_type=refresh_token`): `refresh_token`, plus client authentication. The vendor's examples also send `redirect_uri` (L-0100). Aerie accepts it and, if present, checks it against the client's registered URIs; absent is fine. An optional `scope` may narrow the grant (RFC 6749 §6); it can never widen it.

**Any other `grant_type`**: `unsupported_grant_type`. (`client_credentials` is listed in the decisions in `identity-plan.md`.)

A successful response is `200` with JSON (field names from L-0100, RFC 6749 §5.1):

| Field | Type | Meaning |
|---|---|---|
| `access_token` | string | The access token |
| `token_type` | string | `bearer` (lowercase, as the vendor sends it; clients compare it without regard to case) |
| `expires_in` | integer | Seconds until the access token expires |
| `refresh_token` | string | The new refresh token |
| `created_at` | integer | Unix time, in seconds, when the tokens were issued (vendor field, L-0100) |
| `scope` | string | The granted scopes, separated by spaces (RFC 6749 §5.1; aerie always sends it) |

### Tokens

- **Format.** Opaque random strings, not JWTs. Access tokens start `kbo_`, refresh tokens start `kbr_`, followed by at least 256 bits of randomness, base62-encoded. The vendor's tokens look like JWTs (L-0100), but clients must treat them as opaque, so a different shape is safe (aerie's choice; the lab will confirm). Prefixes let aerie, and secret scanners, recognise them.
- **Storage.** Only a SHA-256 hash of each token is stored, like aerie's existing `kbt_` tokens.
- **Lifetimes** (aerie's defaults, each a site setting):

  | Item | Default | Allowed range | Basis |
  |---|---|---|---|
  | Authorization code | 300 s | 60 to 600 s | L-0105 caps at 600 |
  | Access token | 3600 s | 300 to 7200 s | L-0105 default 3600; L-0100 examples show 7200 |
  | Refresh token, idle | 90 days | 1 to 365 days | L-0105 default 90 days. Measured from the last refresh |
  | Grant, absolute | none | none, or 1 to 730 days | Aerie's choice. If set, the user must consent again after this long |

- A refresh token may be used after its access token has expired (L-0100).

### Refresh rotation and reuse detection

- Every successful refresh invalidates the presented refresh token **and** the access token issued with it, and returns a new pair (L-0100 describes exactly this; RFC 9700 §4.14.2).
- The tokens of one consent form a family. If a refresh token that was already rotated is presented again, aerie treats it as stolen: it revokes the whole family (every access and refresh token of that grant, but not the user's consent) and answers `invalid_grant`. The event is audited. The client must send the user through the authorize flow again.
- Two refreshes racing with the same token: the first wins, the second triggers reuse detection. Aerie does not allow a grace window (aerie's choice; clients the lab runs will show if one is needed).
- A user has at most 25 active grants per client (aerie's choice). Creating a 26th revokes the least recently used.

### Revocation

`POST /rest/oauth2/latest/revoke` (RFC 7009, L-0111; aerie's own addition):

- Form parameters `token` and optional `token_type_hint` (`access_token` or `refresh_token`), with client authentication as at the token endpoint.
- Revoking a refresh token revokes its whole family. Revoking an access token revokes that token only.
- The answer is `200` with an empty body whether or not the token existed or was already revoked, so the endpoint cannot be used to probe tokens. A token that belongs to another client is ignored silently.
- Bad client authentication: `401` with `invalid_client`.

Users and administrators can also revoke without the client: the user's "Connected applications" page, and disabling or deleting the client.

### Server metadata

`GET /.well-known/oauth-authorization-server` (RFC 8414, L-0112; aerie's own addition) returns JSON with `issuer` (the site base URL), `authorization_endpoint`, `token_endpoint`, `revocation_endpoint`, `scopes_supported`, `response_types_supported` (`["code"]`), `grant_types_supported` (`["authorization_code","refresh_token"]`), `code_challenge_methods_supported` (`["S256"]`), `token_endpoint_auth_methods_supported` (`["client_secret_post","client_secret_basic","none"]`) and `revocation_endpoint_auth_methods_supported` (the same three). No authentication. With a context prefix the document is served at the site root as RFC 8414 §3 describes.

### Errors

**At the authorize endpoint**, once client and redirect URI are known to be good, errors are sent as query parameters on the redirect (RFC 6749 §4.1.2.1): `error`, optional `error_description` (aerie's words), and `state` if the request had one. Codes used: `invalid_request`, `unauthorized_client`, `access_denied`, `unsupported_response_type`, `invalid_scope`, `server_error`, `temporarily_unavailable`.

**At the token and revoke endpoints**, errors are JSON (RFC 6749 §5.2) with `error` and optional `error_description`, not the `RestError` shape of `/rest/api`:

| `error` | Status | When |
|---|---|---|
| `invalid_request` | 400 | Missing or repeated parameter, malformed value, PKCE verifier missing or of the wrong shape |
| `invalid_client` | 401 | Unknown or disabled client, wrong secret, public client trying a confidential-only feature. If the client used HTTP Basic, the response carries `WWW-Authenticate: Basic` |
| `invalid_grant` | 400 | Code or refresh token unknown, expired, already used, issued to another client; `redirect_uri` mismatch; PKCE verifier does not match; the user has been disabled |
| `unauthorized_client` | 400 | The client may not use this grant type |
| `unsupported_grant_type` | 400 | Grant type not offered |
| `invalid_scope` | 400 | A refresh asked for scopes beyond the grant |

The vendor's error bodies for these endpoints are not published. RFC 6749 is what OAuth client libraries expect, so it is what aerie sends. The lab's `auth` suite will compare.

### Errors on protected calls

When a `/rest/api` call carries a bearer token (personal access token or OAuth) that aerie does not accept (RFC 6750 §3, L-0109):

- Unknown, expired or revoked token: `401` with `WWW-Authenticate: Bearer error="invalid_token"` and the `RestError` body.
- Valid token, but its scope does not cover the operation: `403` with `WWW-Authenticate: Bearer error="insufficient_scope", scope="<needed scope>"` and the `RestError` body.
- Bearer tokens are accepted only in the `Authorization` header. A token in the query string or form body is ignored (RFC 9700 §4.3.2, RFC 6750 §2).

### Scopes

Scope names, what they cover and which scopes each implies come from L-0100 and L-0104. Whatever scopes a token has, it can never do more than the user who granted it (L-0100, L-0104).

| Scope | Implies | Data Center meaning | In aerie |
|---|---|---|---|
| `READ` | `READ` | View what the user can view, and the user's own profile | Every compatible `GET` and `HEAD` that is not an administration operation |
| `WRITE` | `READ`, `WRITE` | Create, change and delete what the user can change, including spaces, and the user's own profile | Also every compatible `POST`, `PUT` and `DELETE` that is not an administration operation |
| `ADMIN` | `READ`, `WRITE`, `ADMIN` | Most site administration | Also administration operations of the compatible surface, if the user is an aerie administrator |
| `SYSTEM_ADMIN` | all four | All site administration, including backups and imports | Same as `ADMIN` in aerie. Aerie has one administrator level, so `SYSTEM_ADMIN` adds nothing over `ADMIN` |
| `READ_ALL`, `MANAGE_SUBSCRIPTIONS`, `JSM_KB` | | Special-purpose scopes for service-to-service use and a sibling product (L-0104, L-0107) | Not offered. Asking for them is `invalid_scope`. They cannot be selected when registering a client |

Rules:

- `scope` lists names separated by spaces (RFC 6749 §3.3). Order and duplicates do not matter. The token's effective scope set is the union of what each named scope implies.
- A client may only ask for scopes it was registered with. Registering a client with `ADMIN` or `SYSTEM_ADMIN` is allowed; a non-administrator who authorizes it gets a token whose admin scopes simply never pass the user check.
- Each compatible operation needs exactly one scope. The rule is mechanical: `GET` and `HEAD` need `READ`, other methods need `WRITE`, and operations marked administrative need `ADMIN`. No T1 operation is administrative. Later tiers mark administrative operations in the contract (`x-aerie-scope: ADMIN`); an operation with no such mark follows the method rule.
- `GET /rest/api/user/current` and the server information call need `READ`.
- After the scope check passes, the normal permission check runs as the user: space roles (`Reader`, `Contributor`, `Supervisor`, `Owner` in aerie), group membership, sensitivity rules, the hiding rule from `common.md`. Scope can only remove access, never add it.
- The `/rest/oauth2` endpoints themselves need no scope.

How aerie's roles line up, for builders: `READ` admits whatever a `Reader` could do through the compatible surface; `WRITE` admits the further actions a `Contributor`, `Supervisor` or `Owner` holds; `ADMIN` admits what `isAdmin` holds. The role decides; the scope only filters.

## Audit

Aerie's audit log (`/api/v1/admin/audit`, as-is) gets these events. Each records the actor, the client id where there is one, the source address and the outcome. No event ever records a secret, a token or a code, only a token's prefix and id.

| Action | When |
|---|---|
| `oauth.client.create`, `oauth.client.update`, `oauth.client.secret.rotate`, `oauth.client.delete` | Client administration |
| `oauth.consent.grant`, `oauth.consent.deny` | The consent screen |
| `oauth.grant.revoke` | A user, administrator or client revoked a grant |
| `oauth.token.issue` | A code was exchanged |
| `oauth.token.refresh` | A refresh succeeded. May be sampled or rolled up per grant per hour (aerie's choice), since well-behaved clients refresh often |
| `oauth.token.reuse` | A rotated refresh token or used code was presented again; the family was revoked |
| `auth.basic.refused` | Basic was refused: bad password, codes on, password Basic turned off, or locked out. The reason is recorded |
| `token.use.denied` | A token was refused for scope, expiry or revocation. May be rolled up like refresh |

## Open questions for the lab

1. Does any vendor client parse access tokens as JWTs, or otherwise depend on their shape?
2. Do vendor clients send `client_id` and `client_secret` in the query string, the body, or HTTP Basic? Aerie accepts all three; the lab should record which ones are used.
3. Does any client need `WWW-Authenticate: Basic` on a `401` from `/rest/api` before it sends credentials?
4. Does any client refresh concurrently with the same refresh token, so that reuse detection would lock it out?
5. With a token in the Basic password field, what username do clients send? Aerie requires the owner's username.
6. Do any clients use `code_challenge_method=plain`?
