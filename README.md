# aerie-spec

The spec room for [Aerie](https://github.com/synssins/aerie), a self-hosted knowledge base. This repository turns the public record (published API documentation, published OpenAPI files and open standards) into written behaviour specs and OpenAPI contracts, in our own words. Aerie is built to these contracts and nothing else.

## The three repositories

Aerie is redeveloped under a clean-room method. Spec writers and builders never share a repository.

| Repository | Role | Who works in it |
|---|---|---|
| **aerie-spec** (this one) | Written specs, API contracts, the provenance ledger | Spec writers |
| [aerie](https://github.com/synssins/aerie) | The implementation, built only from tagged releases of this repository | Builders |
| [aerie-compat-lab](https://github.com/synssins/aerie-compat-lab) | Runs vendors' published clients against the aerie container image | Lab writers |

Only two things reach the builders: tagged contract releases from here, and black-box results from the lab. [CLEAN_ROOM.md](CLEAN_ROOM.md) has the full policy.

## What the contracts cover

| Contract | Surface | Notes |
|---|---|---|
| `contracts/aerie/openapi.yaml` | Aerie's own API, `/api/v1` | Starts as an as-is export of the current app, then evolves here |
| `contracts/datacenter/openapi.yaml` | Confluence Data Center compatible API, `/rest/api` and `/rest/oauth2` | One contract for Data Center 9.0 and later: the published specs only add operations from 9.0.1 (111) to 11.1.0 (183) |
| `contracts/cloud/v1/openapi.yaml` | Confluence Cloud v1 compatible API, `/wiki/rest/api` | After Data Center |
| `contracts/cloud/v2/openapi.yaml` | Confluence Cloud v2 compatible API, `/wiki/api/v2` | After Data Center |

## Layout

| Path | What |
|---|---|
| [CLEAN_ROOM.md](CLEAN_ROOM.md) | What may and may not come into this repository, and how specs are written |
| [ledger/references.csv](ledger/references.csv) | Every public reference read: address, version, date, SHA-256, who read it, what it informed |
| [contracts/](contracts/README.md) | The OpenAPI contracts |
| [behaviour/](behaviour/README.md) | Written behaviour specs, one file per area |
| [samples/](samples/README.md) | Our own sample documents for the storage format and ADF round-trip tests |
| [questions/](questions/README.md) | Questions from builders, answered by contract changes |

## Contract conventions

- **OpenAPI 3.0.3, YAML.** Atlassian publishes its specs as OpenAPI 3.0.x, so client generators in the lab read ours and theirs the same way.
- **`x-aerie-sources`** on every operation and schema: the ledger IDs it was written from.
- **`x-aerie-since`** on every Data Center operation: the first Data Center version whose published spec includes it.
- **`x-aerie-tier`** on every compatible operation: `T1`, `T2` or `T3`, matching the roadmap.
- **Our own words.** Only functional facts come from vendor sources: paths, parameter and field names, types, enum values, status codes and header names. Descriptions and examples are written here.

## How a change flows

1. Open a pull request here with the contract or behaviour change and its ledger rows.
2. One approving human review, then merge.
3. Tag a release, `contracts-vN`. The release bundle is `contracts/` and `behaviour/` plus `SHA256SUMS`.
4. In `aerie`, a pull request replaces its `contracts/` folder with the release, unchanged, and the code follows.

A builder who needs an answer the contract doesn't give files it in `questions/`. The answer comes back as a contract change, never as a pasted vendor document.

## Status

Phase 1 of the roadmap has not started. First tasks:

- [ ] Receive the as-is export of aerie's API from a build session and commit it as `contracts/aerie/openapi.yaml`
- [ ] Write the Data Center T1 behaviour specs: content, spaces, search and CQL, attachments, labels, versions, users
- [ ] Write `contracts/datacenter/openapi.yaml` for T1
- [ ] Write the storage format behaviour spec and the first samples
- [ ] Tag `contracts-v1`

## License

[Apache License 2.0](LICENSE).
