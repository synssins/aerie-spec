# Q-0035: Largest group, largest merge, and the model's input limit

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

How many documents can a group hold, how many can be merged at once, and what happens when the documents to merge are larger than the model accepts?

## Answer

- A group holds 2 to 10 documents; a larger set keeps the 10 sharing the most tags and leaves the rest ungrouped. Ten is what a reviewer can compare in one block.
- A merge proposal covers 2 to 6 near-duplicates; the owner's example is four or five. More gets no proposal and the reviewer is told to move some out.
- If the documents together exceed the existing `maxInputChars`, no proposal is made and the review says so. Aerie does not split, summarise or merge in rounds: each of those would let the model drop content without the reviewer seeing all sources side by side. The reviewer can move documents out and try again, keep them separate, or merge drafts by hand with the existing `import.merge`.
- The existing import limits (files per batch, sizes, concurrency) are unchanged.

Written in `behaviour/import-grouping.md` (Groups, Near-duplicates and merge proposals).
