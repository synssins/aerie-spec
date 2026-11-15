# Q-0008: CQL dates and loose matching

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/cql-time-zone-and-fuzzy.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Which time zone are CQL dates read in, and on which day do weeks start? How loose is a word followed by `~`?

## Answer

The build room's reading: **stands**.

The published behaviour reads dates in the caller's configured zone, falling back to the server's (L-0007). Aerie has neither, so UTC is the honest server zone, and weeks start on Monday. When aerie gains a per-user or site zone, that zone applies. Loose matching is not defined in the public record; aerie's one-letter rule stands as its own choice.

Written in `behaviour/search-cql.md` (Values, Functions, Text matching).
