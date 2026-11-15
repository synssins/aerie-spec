# Q-0012: Who sees a space made through the compatible surface

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/space-visibility-on-create.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Which aerie visibility does `POST /rest/api/space` give a new space?

## Answer

The build room's reading: **changes**.

The contract has a separate operation for a space only its creator can see, so a space made with `POST /rest/api/space` must be visible to more people, or the two operations would be the same. An integration expects other users to find the space. It is therefore made Signed in: everyone signed in reads it. `_private` stays Private.

Written in `behaviour/spaces.md` (Creating); `createSpace` description.
