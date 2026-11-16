# Q-0021: Is password Basic refused for the emergency administrator account?

Raised by: the build room, while pinning `contracts-v6`.
Status: answered in pull request `phase1-rulings-6`.

## Where

Data Center compatible surface: HTTP Basic with a password, for aerie's emergency administrator (break-glass) account.

## Question

`behaviour/common.md` and `behaviour/identity-plan.md`, as written for Q-0020, say password Basic is always refused for administrators and the emergency administrator account. `common.md` gave the reason that the emergency account always has codes. That reason is wrong: in aerie the emergency account never uses sign-in codes, by design. Its sign-in is written to the audit log and alerts the administrators. Before this, aerie allowed it password Basic, with an audit entry and an alert. Is password Basic refused for it?

## Answer

Yes. The refusal stands.

- Password Basic for the emergency account gets `401`, the same response and body as a wrong password and as the other refused cases.
- The audit log records the reason `break-glass`.
- Administrators are still alerted, as for any sign-in to that account.
- Its sign-in in the browser does not change.

Reason: the emergency account skips the second factor so that it still works when nothing else does. It must not also be usable from scripts with only a password.

The rule does not change. Only its stated reason was wrong. `common.md` now says the emergency account is refused because it has no second factor, not because it has codes. The lab case `auth.basic.codes_exempt.admin_refused` in `identity-plan.md` already expects `401` and stays as it is. Exempting the emergency account still answers `409` (question Q-0020).

Written in `behaviour/common.md` (Accounts with sign-in codes). This text goes out with the next contract release; no release is needed for it alone.
