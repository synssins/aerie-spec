# Q-0015: Who may read users and group memberships

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/user-directory-visibility.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Who may read another user, list users and read another user's groups? Are deleted and system accounts listed?

## Answer

The build room's reading: **stands**.

The build room's reading stands: aerie's rule for looking people up (administrators, space owners, anyone who may edit pages somewhere), own record always; groups of others for administrators only; anonymous `401` for lists. Deleted accounts and the emergency administrator are neither listed nor found; deactivated accounts are listed.

Written in `behaviour/users.md` (Who may look people up); `listUsers` and `getUserGroups` descriptions.
