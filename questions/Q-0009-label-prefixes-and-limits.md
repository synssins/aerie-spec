# Q-0009: Label prefixes and aerie's tag limits

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/label-prefixes-and-limits.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Should aerie hold personal (`my`) and team labels, and widen its tags from 50 characters and 20 per page to the 255 characters in the contract?

## Answer

The build room's reading: **stands**.

Personal and team labels are T2; in T1 only `global` labels exist, as the build room did. The 255-character limit in the contract was aerie's own choice, not from a source: no pinned specification sets a label length or count. Aerie's limits (50 characters, 20 per item, all-or-nothing on refusal) stand, and the contract now says so.

Written in `behaviour/labels-properties.md`; `LabelWrite.name` and `addContentLabels` in the contract.
