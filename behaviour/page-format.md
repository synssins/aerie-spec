# Format: cutting run-together items and fixing spelling

Sources: L-0025

Aerie's own feature, not a vendor-compatible surface. It extends the existing format call (`POST /api/v1/pages/{id}/format`, `page.format`), which asks the model to give each part of a page a role and answers with a preview and its diff. Two optional fields are added to each role (`FormatRoleDto`), in the request's and in the answer's `roles`:

- `split`: where to cut a paragraph that runs several "Label: value" items together.
- `fix`: spelling and grammar corrections.

Both are in `contracts/aerie/openapi.yaml`, marked `x-aerie-status: planned` until the build pins the release. The rest of the format call is unchanged.

## What does not change

- The model only labels parts and proposes cuts and corrections. Aerie applies them to the page's own content and checks each one.
- The every-word guard stays: the result must keep every word of the page. It now compares the result with the page as corrected, so an allowed correction does not trip it.
- Nothing is saved until the person accepts the preview.
- Format does not fill the structured page fields (applies to, owner, review interval). Those need an inventory application, a person or group, and a number of months, which needs matching and a confirm step.

## 1. Cutting run-together items (`split`)

A paragraph such as "**Applies to:** `kv-platform-prod`. **Owner:** Cloud Platform Engineering. **Last verified:** last week." holds three items. The model gives that part the role `paragraph` and, in `split`, the exact text each item after the first starts with ("Owner:", "Last verified:").

- Aerie cuts the page's own inline content at each of those places. Each piece becomes its own paragraph, with its marks (bold, code, links) kept.
- A cut whose text is not found at the start of a word in that paragraph is ignored. The rest of the result still applies.
- Only parts whose role is `paragraph` are cut. `split` on any other role is ignored.
- There is no code for particular labels. Any "Label: value" run is handled the same way.

## 2. Spelling and grammar (`fix`)

Each part may carry `fix`: a list of `{ "from": "Mofdnitoring", "to": "Monitoring" }`. The model is told to correct only clear mistakes and never to reword, change meaning, facts, numbers or names, or touch code, commands, paths, host names, identifiers, product names, URLs or quoted error messages. A word it is unsure about stays as written.

### Allowed corrections

A correction is allowed when all of these hold:

- Both sides have the same numbers.
- Each side has one to three words, and the sides differ by at most one word.
- Both sides start with the same letter.
- Only a few letters change: at most a quarter of the letters, and at least one is always allowed. A swap of two neighbouring letters counts as one change.

A change of capitals only is also allowed.

Examples: `Mofdnitoring` to `Monitoring`, `teh` to `the`, `recieve` to `receive`, `a` to `an`, `dont` to `don't`.

### Refused corrections

A correction that changes content refuses the whole result. The call answers `422` with a problem body, nothing is changed, and the refusal is audited as `content-changed`. Content changes are:

- another number (`40` to `50`);
- another word (`Owner` to `Manager`), the opposite (`enabled` to `disabled`), or a rewording (`restart` to `reboot`);
- words added or removed beyond the limit above.

### Places never corrected

Corrections are not made in these places. The text stays as written, and the rest of the result still applies:

- code blocks and inline code;
- text inside double quotes;
- a word written like an identifier, host name, path or version: one that holds a hyphen, underscore, dot, slash or digit, or a capital letter after its first letter.

### Showing and sending back

- Every correction shows as a changed line in the preview's diff before the person accepts it.
- Corrections travel back in `roles`. Rebuilding a preview from roles sent back needs no model call.
- Corrections sent back are checked by the same rules. A sent-back correction that changes content is `422`, audited as `content-changed`.

## Lab cases

The lab points the AI settings at its stub model.

- `page.format.split`: a paragraph with three bold "Label:" items and the stub's `split` for the second and third gives three paragraphs; bold and inline code are kept; every word is kept.
- `page.format.split.bad_cut`: a cut text not found, or found only inside a word, is ignored; the other cuts apply.
- `page.format.split.not_paragraph`: `split` on a heading or list item changes nothing.
- `page.format.fix`: `Mofdnitoring` comes back `Monitoring`, and the diff shows the changed line.
- `page.format.fix.skipped`: the same misspelling in inline code, a code block, a double-quoted message and a host name stays as written.
- `page.format.fix.allowed`: `teh`, `recieve`, `a` to `an`, `dont` and a capitals-only change are all made.
- `page.format.fix.refused`: a fix of `40` to `50`, `Owner` to `Manager`, `enabled` to `disabled` or `restart` to `reboot` is `422`, nothing is changed, and an audit entry records `content-changed`.
- `page.format.fix.sent_back`: rebuilding from the answer's `roles` makes no model call and gives the same preview; the same roles with a number-changing fix are `422`.
