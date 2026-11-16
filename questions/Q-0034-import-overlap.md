# Q-0034: How overlap between documents is judged

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

Where several files cover almost the same thing, the import suggests merging them. How is "almost the same" judged?

## Answer

- Aerie measures it, without the model. Both texts are normalised (lower case, punctuation and extra spaces removed) and split into runs of 5 words; the overlap is the share of the smaller document's runs that also occur in the larger.
- Two documents overlapping by 0.5 or more are near-duplicates; near-duplicates of near-duplicates join one set. Only documents in the same group are compared.
- 0.5 catches documents rewritten over the years that still keep half their wording, and leaves out documents that only share a subject. It is a constant in this first version.

Reason: a measured number is the same on every run, can be tested with fixed files, and cannot be talked into a merge by text inside a file.

Left to the owner: the 0.5 threshold, if real shares show it merges too much or too little.

Written in `behaviour/import-grouping.md` (Near-duplicates and merge proposals).
