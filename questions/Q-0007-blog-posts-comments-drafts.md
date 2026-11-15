# Q-0007: Blog posts, comments and unpublished drafts

Raised by: the build room, after routing all 52 T1 operations (aerie repository, `docs/spec-questions/blog-posts-comments-drafts.md`, ledger L-0026).
Status: answered in pull request `phase1-rulings-2`.

## Question

Aerie holds no blog posts, no comments and no drafts of never-published content. Should it gain them, and what do the operations answer until then?

## Answer

The build room's reading: **stands**.

Outside T1. Blog posts and comments are T2; unpublished drafts are T3. Each needs a new aerie data model, and an aerie draft (unsaved edits to an existing page) is a different thing from the vendor's. Until then the build room's reading is what an integration sees: empty lists, `400` on create, `404` for `status=draft`.

Written in `behaviour/content.md` (What T1 leaves out, Trashing and purging); `deleteContent` and `deleteContentVersion` descriptions.
