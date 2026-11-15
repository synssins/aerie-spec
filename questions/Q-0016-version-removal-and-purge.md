# Q-0016: Removing a version, and purging from the trash

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/version-removal-and-purge.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Must version removal, purge and emptying the trash destroy the data, or may aerie keep it out of reach of the compatible surface?

## Answer

The build room's reading: **stands**.

An integration can only observe the compatible surface: the version or item is gone for good from every read, and its number or id is never reused. That is what aerie does, so the build room's reading stands. The data stays in aerie's own history or administrators' bin, and an administrator emptying that bin is the step that destroys it. The spec now says so instead of "for good".

Written in `behaviour/content.md` (Trashing and purging, Versions), `behaviour/spaces.md` (Trash); `deleteContentVersion`, `deleteAttachmentVersion` and `emptySpaceTrash` descriptions.
