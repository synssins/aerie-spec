# Q-0037: Accepting a merge, conflicts, and keeping the sources

Raised by: the spec room, while specifying the owner's request for related documents in the AI import.
Status: answered in pull request `phase1-features-2`.

## Where

Aerie's own API under `/api/v1`: the existing `import.*` calls and the planned `import.group.*` calls. Not a compatible surface.

## Question

How does a reviewer accept, refuse or adjust a proposed merge, how are conflicts settled, and how do the source files stay traceable?

## Answer

- The proposal shows the merged page next to its sources, which document each section came from, every conflict with each document's wording, and statements found in no source.
- Conflict defaults are picked by aerie from the dates (question Q-0036), never by the model. The reviewer can change any choice.
- Accepting uses the existing `import.merge` with the proposal, the choices and the proposal version. It produces one merged draft; a page is saved only when that draft is accepted through the existing `import.accept`. A missing choice is `422`; a stale proposal is `409`.
- Keep separate drops the proposal; moving a document out rebuilds it from the rest.
- A page made from a merge always has every source file attached, whatever `attachOriginals` says, and its first version's change note lists the source files with their dates. The source drafts point to the page.

Reason: nothing is merged or saved without the reviewer, and anyone reading the page later can open the original files.

Written in `behaviour/import-grouping.md` (Accepting a merge); `ImportMergeRequest` and `ImportMergeProposalDto` in `contracts/aerie/openapi.yaml`.
