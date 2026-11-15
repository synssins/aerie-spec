# Q-0011: Build numbers in server information

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/server-information-build-numbers.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

What build numbers and build date does aerie report for each version it may present?

## Answer

The build room's reading: **stands**.

No ledgered source lists the vendor's build numbers, so the table `common.md` promised is replaced by the build room's rule: settings, and when unset, major x 1,000,000 + minor x 1,000 + patch. That keeps numbers growing with the version, which is what clients that compare them rely on. The build date is a setting with a fixed default.

Written in `behaviour/common.md` (Server information); `getServerInformation` description.
