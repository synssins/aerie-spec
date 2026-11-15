# Q-0013: Title length

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/title-length.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Should aerie widen titles from 200 to the 255 characters in the contract?

## Answer

The build room's reading: **stands**.

The 255 in the contract was aerie's own choice; no pinned specification sets a title length. Aerie's 200 stands and the contract now says 200. A longer title gives `400`.

Written in `behaviour/content.md` (The content object); `ContentWrite.title` in the contract.
