# Q-0006: Attachments on folders

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/attachments-on-folders.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

May a client add an attachment to a folder, or move one onto a folder? Aerie's own upload refuses folders; the build room answers `400`.

## Answer

The build room's reading: **changes**.

A folder shows as `type: page`, and Q-0002 rules that nothing on the compatible surface tells it apart from a page. A client that uploads to a page and gets `400` has no way to understand why. So folders accept new attachments and moved attachments, like pages. Aerie already keeps attachments on folders that were pages, so its data allows it; its web app must list them on folders too.

Written in `behaviour/attachments.md` (Uploading, Moving); `createAttachments` description.
