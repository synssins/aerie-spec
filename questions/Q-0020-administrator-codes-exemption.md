# Q-0020: Can an administrator be exempted from sign-in codes?

Raised by: the build room (identity slice 1) and the lab case `auth.basic.codes_exempt.ok` on image `4034a25`.
Status: answered in pull request `phase1-rulings-5`.

## Where

Aerie's own API: `PUT /api/v1/admin/users/{id}/codes/exempt` (`user.codes.exempt`). It affects password Basic on the Data Center compatible surface.

## Question

Exempting an administrator or the emergency administrator (break-glass) account answers `409` ("An Admin always uses codes."), as aerie did before the identity work. The lab expected `204`, and then `200` for Basic with the password, because `identity-plan.md` said "the same administrator, exempted, gets `200`". The spec contradicted itself:

- `identity.md` and `common.md` said administrators are required to use codes.
- Both also said exempted accounts may use the password in Basic, without saying whether an administrator can be exempted.
- The lab case assumed an administrator could be exempted.
- The aerie contract listed only a generic `409`.

## Answer

No. Administrators and the emergency administrator account cannot be exempted. Exempting one answers `409`, and nothing changes. The build room's behaviour stands. The lab case and the spec change.

Reason: an exemption would let anyone holding an administrator's password use the API with no second factor. Codes are required for administrators to prevent exactly that. This is aerie's own API and policy, so no client of the compatible surface is affected.

Consequences:

- Password Basic is always refused (`401`) for administrators and the emergency administrator account. They use a token, which was created inside a session that passed the code.
- An exemption has no effect while the account is an administrator. An exempted user who is made an administrator must use codes from then on. Without this, the rule could be avoided by exempting an account before promoting it.
- Ending an exemption (`exempt: false`) is always allowed.
- The lab case `auth.basic.codes_exempt.ok` now uses a user who is not an administrator, has codes turned on, and is then exempted. The new case `auth.basic.codes_exempt.admin_refused` checks the `409`.

Written in `behaviour/identity.md` and `behaviour/common.md` (Accounts with sign-in codes); `behaviour/identity-plan.md` (slice 1 lab cases); `user.codes.exempt` in `contracts/aerie/openapi.yaml` (description and an explicit `409`).
