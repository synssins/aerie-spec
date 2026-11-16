# Q-0033: How the import decides documents are about the same system

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

Many dropped files must be shown in groups when they cover the same system. Who decides, on what evidence, and can a document be in several groups?

## Answer

- Aerie groups, not the model. Two documents are about the same system when they share a `system` or `application` tag after aerie matched the suggested tags against the inventory and existing tags (question Q-0038). An inventory link counts as the same tag whatever the spelling.
- Each document is in at most one group, the group of its first such tag. One place per document keeps the review simple and makes a merge unambiguous.
- Grouping runs once, after every document in the batch has finished drafting, and only within that batch.
- The reason shown is written by aerie from the shared tags, not by the model, so text in a file cannot put words in the review.

Reason: the model already reads each document to draft it, so asking it for tags costs nothing extra; turning tags into groups with a plain rule makes grouping explainable, testable and immune to instructions hidden in files.

Written in `behaviour/import-grouping.md` (Groups).
