# Behaviour specs

Written specs of how aerie behaves, one file per area, in our own words. Each file starts with the ledger IDs it was written from.

Planned files, in the order the roadmap needs them:

| File | Covers |
|---|---|
| `common.md` | Rules every Data Center compatible operation shares: paging, expansion, links, errors, the XSRF header, server information, long tasks |
| `content.md` | Pages, folders, blog posts, comments, versions, status, the tree |
| `spaces.md` | Spaces, keys, permissions, roles |
| `search-cql.md` | Search and the supported CQL subset |
| `attachments.md` | Attachments and their versions |
| `labels-properties.md` | Labels with prefixes, content and space properties |
| `users.md` | Reading users and the groups they are in |
| `representations.md` | Storage format, ADF, view, the opaque node |
| `identity.md` | OAuth 2.0, OIDC, LDAP, tokens, basic auth, scopes |
| `restrictions.md` | Page-level read and update restrictions |
| `webhooks.md` | The event outbox and webhook delivery |
| `migration.md` | The Data Center importer and what it preserves |

Written so far (Data Center T1): `common.md`, `content.md`, `spaces.md`, `search-cql.md`, `attachments.md`, `labels-properties.md` (labels only), `users.md`, `representations.md` (storage format only). Each ends with the open questions the lab should settle.
