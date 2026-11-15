# Storage format samples

Our own sample bodies for the storage format round trip described in [behaviour/representations.md](../../behaviour/representations.md). Each file is a storage body exactly as a client would send it in `body.storage.value`. Every file is written here and has a row in [the ledger](../../ledger/references.csv).

For every sample in this folder, the body aerie returns when read back as `storage` (`B'`) must be equivalent to the file, under the equivalence rules in the round trip guarantee. Samples whose `B'` differs from the input will carry a matching `*.expected.xml` file; none do yet.

| File | What it exercises |
|---|---|
| `01-text-blocks.xml` | Headings, inline marks, nested lists, a block quote, a table with `colspan`, a rule, centred text, an entity |
| `02-links-images.xml` | `ac:link` to a page (same space and other space), an attachment with a rich link body, a same-page anchor and a user; `ac:image` from an attachment and from a URL; a plain `a` link |
| `03-tasks-layout.xml` | A two-section page layout, a task list with both statuses, emoticons, a placeholder |
| `04-macros-opaque.xml` | A rendered `info` macro with a rich body, a `code` macro whose CDATA holds characters that must survive byte for byte, an unknown add-on macro and an unknown namespaced element, both kept as opaque nodes |

ADF samples and the expected aerie editor JSON for each sample come later: ADF with the Cloud contracts, editor JSON once the build room's as-is export of aerie's own API is committed.
