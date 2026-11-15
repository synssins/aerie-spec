# Clean-room policy: spec room

This repository turns the public record into written specs and API contracts, in our own words. Builders never work here; they receive tagged releases.

## The three repositories

| Repository | Role |
|---|---|
| `aerie-spec` (this one) | Spec room: written specs, contracts, the provenance ledger |
| `aerie` | Build room: implementation from tagged contract releases only |
| `aerie-compat-lab` | Lab: vendors' published clients run against the aerie container image |

## May be read

- Vendors' public API documentation and published OpenAPI files
- Open standards and RFCs
- Public documentation of community client libraries
- Black-box results from `aerie-compat-lab`
- Aerie's own code and documentation (for the as-is contract of aerie's own API)

## Never allowed in any repository

- Leaked or proprietary source code
- Decompiled or disassembled binaries
- Confidential information from a current or former employer
- Copied UI assets, screenshots or trademarks of another product
- Customer code or customer data

## Writing rules

1. **No copies of vendor material.** Do not commit vendor documents or vendor spec files. Record each one in `ledger/references.csv`: address, version, retrieval date, SHA-256, who read it, what it informed.
2. **Functional facts only.** From vendor sources take only paths, parameter and field names, types, enum values, status codes and header names. Write every description, example and behaviour statement in our own words.
3. **Cite sources.** Each contract operation and schema names its ledger rows in an `x-aerie-sources` list. Each behaviour spec lists them at the top.
4. **Our own samples.** Sample documents in `samples/` are written here and listed in the ledger.

## Releases

Tag `contracts-vN`. The release bundle is `contracts/` and `behaviour/` plus `SHA256SUMS`, laid out as `aerie`'s `contracts/` folder expects. A release is never changed after it is tagged.

## Sessions and review

- A spec session (person or AI agent) does not write implementation code in `aerie`. Governance files there are the only exception.
- Every change lands by pull request with one approving human review.

## If something slips through

Stop, and do not merge. Remove the material, record what happened in the ledger's incident log (`ledger/incidents.md`), and have the affected spec text rewritten by someone who has not seen the material. Tell the build room which released contracts were affected.
