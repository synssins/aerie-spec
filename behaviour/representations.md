# Representations: storage format, view and the opaque node

Sources: L-0001, L-0007, L-0014, L-0015, L-0020

How aerie accepts, stores and returns page bodies on the Data Center compatible surface, and the body conversion operation. ADF (the Cloud document format) is added to this file when the Cloud contracts start. The samples in `samples/storage/` illustrate every rule here.

## Body representations

A content `body` holds one entry per representation that was expanded. Each entry is `{ "value": string, "representation": string }`.

| Representation | Direction | What it is |
|---|---|---|
| `storage` | read and write | The XHTML-based storage format. The source of truth |
| `editor` | read and write | The form the web editor works on. Aerie returns its own editor markup; see below |
| `view` | read only | HTML ready to show inside the site's page chrome |
| `export_view` | read only | HTML with absolute links, for showing outside the site |
| `styled_view` | read only | `export_view` wrapped in a full HTML document with inline styles |
| `anonymous_export_view` | read only | `export_view` rendered as the anonymous user would see it |

Writing a read-only representation gives `400`.

## Storage format: what a body is

- A storage body is an XML fragment: a sequence of block elements with no single root element and no XML declaration. Aerie parses it by wrapping it in a root element that declares the `ac`, `ri` and `at` prefixes.
- Named HTML entities (`&nbsp;`, `&mdash;`, `&ndash;` and the rest of the XHTML set) are accepted and kept. Aerie writes them back as the same entity or as the character, at its choice; clients must not depend on which.
- Text inside `ac:plain-text-body` and `ac:plain-text-link-body` is CDATA and kept byte for byte.
- A body that is not well-formed XML after wrapping gives `400` on write. Aerie never stores a broken body.
- An empty string is a valid, empty body.

## Elements aerie understands

Plain XHTML:

- Text blocks: `p`, `h1` to `h6`, `blockquote`, `pre`, `hr`, `br`.
- Inline: `strong`, `b`, `em`, `i`, `u`, `s`, `del`, `sub`, `sup`, `code`, `small`, `big`, `span` (with `style` for colour and line-through), `a` (with `href`).
- Lists: `ul`, `ol`, `li`, nested to any depth.
- Tables: `table`, `colgroup`, `col`, `thead`, `tbody`, `tr`, `th`, `td`, with `rowspan` and `colspan`.
- `style` on `p` and headings for `text-align` (`left`, `center`, `right`, `justify`) and margin-left indents.

Prefixed elements (`ac` for structure, `ri` for resource identifiers):

- Links: `ac:link`, with optional `ac:anchor`, holding one resource identifier and an optional body (`ac:plain-text-link-body` or `ac:link-body`). With no resource identifier and an `ac:anchor`, it points at an anchor on the same page.
- Images: `ac:image` holding `ri:attachment` or `ri:url`. Attributes kept: `ac:align`, `ac:border`, `ac:class`, `ac:title`, `ac:style`, `ac:thumbnail`, `ac:alt`, `ac:height`, `ac:width`, `ac:vspace`, `ac:hspace`.
- Resource identifiers:
  - `ri:page` with `ri:content-title` and optional `ri:space-key`,
  - `ri:blog-post` with `ri:content-title`, `ri:posting-day` (`YYYY/MM/DD`) and optional `ri:space-key`,
  - `ri:attachment` with `ri:filename`, optionally holding a `ri:page` or `ri:blog-post` for an attachment on other content,
  - `ri:url` with `ri:value`,
  - `ri:user` with `ri:userkey` (or `ri:username`, which aerie resolves to the user key on write),
  - `ri:space` with `ri:space-key`,
  - `ri:content-entity` with `ri:content-id`,
  - `ri:shortcut` with `ri:key` and `ri:parameter`.
- Emoticons: `ac:emoticon` with `ac:name` set to `smile`, `sad`, `cheeky`, `laugh`, `wink`, `thumbs-up`, `thumbs-down`, `information`, `tick`, `cross` or `warning`.
- Tasks: `ac:task-list` holding `ac:task` items, each with `ac:task-id`, `ac:task-status` (`incomplete` or `complete`) and `ac:task-body`.
- Placeholders: `ac:placeholder`, with optional `ac:type`, shown only in the editor.
- Layouts: `ac:layout` holding `ac:layout-section` elements, each with `ac:type` (`single`, `two_equal`, `two_left_sidebar`, `two_right_sidebar`, `three_equal`, `three_with_sidebars`) and as many `ac:layout-cell` elements as the type has columns (1, 2, 2, 2, 3, 3). When a body uses a layout, `ac:layout` is its only top-level element. A wrong cell count gives `400`.
- Macros: `ac:structured-macro` with `ac:name` and optional `ac:schema-version` and `ac:macro-id`, holding `ac:parameter` elements (each with `ac:name`, text content) and at most one of `ac:plain-text-body` or `ac:rich-text-body`. Which macros aerie renders is aerie's own choice; T1 renders these: `code`, `noformat`, `info`, `note`, `warning`, `tip`, `panel`, `expand`, `toc`, `anchor`, `status`, `children`, `excerpt`, `excerpt-include`, `include`. Every other macro is kept as an opaque node (below).
- Template variables (`at:` prefix) appear only in templates, which are T3. In a page body they are kept as opaque nodes.

## The opaque node

Anything aerie does not understand is kept, not dropped. This is the rule that makes round trips safe.

- An unknown element (any prefix, or an HTML element outside the list above), with everything inside it, is stored as an opaque node: the exact markup, byte for byte, plus its position in the document.
- A known macro name aerie cannot render, and any macro from an add-on, is an opaque node.
- Writing the body back out in `storage` returns the opaque node's markup unchanged, in the same place. Text around it may be re-serialised (attribute order, quoting, self-closing tags), but the node itself is not touched.
- In `view` and the other rendered forms, an opaque node renders as a placeholder box naming the element or macro, with its plain-text content, if any, inside.
- In aerie's own editor, an opaque node is a block that can be moved or deleted but not edited.

## Round trip guarantee

For any well-formed storage body `B` that aerie accepts:

1. Write `B`, then read it back as `storage`: the result `B'` is equivalent to `B`. Equivalent means the same XML infoset after: dropping whitespace-only text between block elements, normalising attribute order, writing empty elements in one form, and replacing entities with the characters they stand for.
2. Write `B'` again: the stored body is byte-identical to the one stored before (`B'` is a fixed point). A successful `PUT` still makes a new version, as `content.md` requires; this guarantee is about the body, not the version chain.
3. Every opaque node in `B` is byte-identical in `B'`.

The samples in `samples/storage/` each state the `B'` aerie must return.

## Rendering for view

- `view` and `export_view` are HTML fragments. Aerie's markup, class names and structure are its own and are not part of the contract beyond: headings keep their level, paragraphs and lists keep their order and nesting, links and images resolve to working URLs, and macros either render or show the opaque placeholder.
- `export_view` uses absolute URLs for every link and image; `view` may use paths.
- `styled_view` is a full HTML document (`<html>`, `<head>` with a `<style>` block, `<body>`).

## The editor representation

- Reading `editor` returns aerie's editor markup for the body. Clients that read `editor` and write it back unchanged must get an equivalent storage body; nothing else about the editor markup is promised.
- Writing `editor` that aerie did not produce is converted on a best-effort basis: known HTML elements map to storage, anything else becomes an opaque node.

## Body conversion

`POST /rest/api/contentbody/convert/{to}`

- Body: `{ "value": ..., "representation": ... }`, the body to convert.
- Path `to`: the target representation.
- Supported pairs: from `storage` to `view`, `export_view`, `styled_view` or `editor`; from `editor` to `storage`. Any other pair gives `400`.
- The conversion is done without a page. Links and macros that depend on a page (relative `ri:page` without a space key, `children`, `toc`) render as if on a top-level page in no space: relative links show as unresolved, page-bound macros render empty. A client can supply context through the `content` field of the body object: `{ "content": { "id": "..." } }` makes the conversion run as if on that page, if the caller can view it.
- `expand`: accepted, applies to `content` in the response when present. Default nothing.
- Result: `200` with `{ "value", "representation" }` in the target representation.

## Open questions

- The storage format sources read so far show `incomplete` as a task status and `ac:task-id`, but not `complete`. Aerie accepts and keeps `complete`; the lab should confirm what real servers write.
- `ac:structured-macro`, `ac:name`, `ac:parameter` and `ac:rich-text-body` appear in the published spec (L-0007). `ac:plain-text-body`, `ac:macro-id` and `ac:schema-version` do not appear in any source read so far; the public page on macro storage markup is still to be found and ledgered. Until then these three names stand as aerie's assumption.
- The allowed values of `ac:align` on images are not in the sources read so far. Aerie keeps any value and renders `left`, `center` and `right`.
