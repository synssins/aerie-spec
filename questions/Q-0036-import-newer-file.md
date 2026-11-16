# Q-0036: How the import decides which file is newer

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

When merged files disagree, the newer file wins by default. How are dates read?

## Answer

In this order:

1. The last-modified date stored inside the file by the program that wrote it, if present and plausible (not in the future, not before 1990).
2. Otherwise the file's last-modified time as the browser reported it on drop, sent by the web app with each file.
3. Otherwise no date.

Dates written in the text are not used: they are often stale, and file content must never steer a decision. When the two newest documents in a conflict have the same date, or one has none, there is no default and the reviewer must choose. Copying a share can reset file-system dates, which is why the date inside the file comes first; each file shows where its date came from.

Written in `behaviour/import-grouping.md` (Dates); `modified` on `import.file.add` and `ImportFileDto.modifiedAt` in `contracts/aerie/openapi.yaml`.
