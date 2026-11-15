# Q-0014: Who sees and manages trashed content

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/trash-permissions.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Which aerie permission do the trash operations use: space administration (Owner) or the space's bin right (Supervisor and up)?

## Answer

The build room's reading: **stands**.

The contract names the permission that governs a space's trash. In aerie, the space's trash is governed by the bin right, not by space ownership, so the build room's reading stands: bin right holders read, restore, purge, list and empty; others get `404` for trashed content and `403` for the space trash.

Written in `behaviour/content.md` and `behaviour/spaces.md` (Trash); `listSpaceTrash`, `emptySpaceTrash` and `deleteContent` descriptions.
