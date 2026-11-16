# Q-0038: What tags the import suggests

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

Each imported document gets suggested tags (system, application, server, topic), reusing existing tags and the application inventory. How?

## Answer

- The model suggests up to 8 tags per document, each with a kind, in the same call that drafts the document.
- Aerie matches them, ignoring case and spacing: application or system names to inventory applications (linked by id, and suggested in "applies to"), server names to servers in inventory installs, and anything equal to an existing page tag to that tag's spelling. Others are new tags, cleaned to aerie's tag rules, or dropped if they cannot be.
- Suggestions are shown on the draft and saved only when the reviewer accepts it, as edited.
- The inventory is only read. The import never creates inventory applications or servers.

Written in `behaviour/import-grouping.md` (Tag suggestions); `ImportItemDto.suggestedTags` and `ImportTagSuggestionDto` in `contracts/aerie/openapi.yaml`.
