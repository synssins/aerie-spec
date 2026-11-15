# Q-0010: Who may create a personal space

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/personal-spaces.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Should any signed-in user create their own personal space, and what visibility does `isPrivate: false` give it?

## Answer

The build room's reading: **stands**.

Personal spaces for every user are T2: they need aerie to have user-owned spaces. The contract already allows `403` when the caller may not have one, so in T1 only administrators may, the space is Private whatever `isPrivate` says, and the key is `~` plus the lower-case username.

Written in `behaviour/spaces.md` (Creating); `createPersonalSpace` description.
