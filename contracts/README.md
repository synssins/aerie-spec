# Contracts

The OpenAPI 3.0.3 contracts aerie is built to, one folder per API surface. Conventions are in the [README](../README.md#contract-conventions).

| Folder | Surface | Status |
|---|---|---|
| `aerie/` | Aerie's own API, `/api/v1` | As-is export of aerie at commit `4cee64d` (ledger L-0025), plus operations not yet built, marked `x-aerie-status: planned` (OAuth 2.0 clients, consent, grants, sign-in settings) |
| `datacenter/` | Data Center compatible, `/rest/api`, `/rest/oauth2` (9.0 and later) | T1 drafted: 52 operations under `/rest/api` and 4 for OAuth 2.0 (`/rest/oauth2/latest/authorize`, `token`, `revoke` and the metadata document), not yet released. T2 and T3 to come |
| `cloud/v1/` | Cloud v1 compatible, `/wiki/rest/api` | After Data Center |
| `cloud/v2/` | Cloud v2 compatible, `/wiki/api/v2` | After Data Center |

Marks used beyond the conventions in the README:

- `x-aerie-extension: true`: an operation aerie adds from an open standard that the vendor does not document (OAuth 2.0 revocation and server metadata). Vendor clients do not need it.
- `x-aerie-since` on the `/rest/oauth2` operations is `9.0.1`, the floor of this contract: the published specs do not list these paths, and the provider documentation places them in every version the contract covers.
- `x-aerie-status: planned` (aerie's own contract): written here ahead of the code, not yet built. The build room removes the mark when the as-is export includes the operation.
