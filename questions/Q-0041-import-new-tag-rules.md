# Q-0041: How a new suggested tag is cleaned in the AI import

Raised by: the build room, against `contracts-v8`.
Status: answered in pull request `import-tag-rules`.

## Where

Aerie's own API under `/api/v1`: the planned `ImportItemDto.suggestedTags` in the AI import, and aerie's existing page tags.

## Question

`behaviour/import-grouping.md` (Tag suggestions) says a new suggested tag is "cleaned to aerie's tag rules", but its lab case expected `month end` to become `month-end`. Does the import turn spaces into hyphens, or keep aerie's existing tag rules?

## Answer

A new suggested tag keeps aerie's existing tag rules, with nothing added for the import:

- trimmed;
- lower case;
- runs of spaces collapsed to one;
- 1 to 50 characters;
- no commas.

Spaces are allowed and nothing turns them into hyphens, so "Month End" becomes `month end`, never `month-end`. A tag that still breaks these rules after cleaning is dropped.

Matching a suggestion to an existing tag still ignores case, spaces, hyphens and underscores, so `Month-End` matches an existing `month end` and reuses that spelling.

Reason: one set of tag rules for every way a tag is made, so an imported page's tags look and behave like tags typed by hand.

Written in `behaviour/import-grouping.md` (Tag suggestions, lab cases `import.group.tags` and `import.group.tags.invalid`); question Q-0038.
