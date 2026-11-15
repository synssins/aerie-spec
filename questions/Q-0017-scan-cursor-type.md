# Q-0017: Type of the scan cursor fields

Raised by: the compatibility lab, run against all 52 T1 operations with clients generated from each pinned published file.
Status: answered in pull request `phase1-rulings-3`.

## Where

Data Center compatible surface: `GET /rest/api/content/scan` (`scanContent`). The same page shape serves `GET /rest/api/space/{spaceKey}/trash` (`listSpaceTrash`).

## Question

Our contract and aerie send `nextCursor` and `prevCursor` as strings. Clients generated in strict mode from the 9.0.1 published file reject the response, because that file declares both fields as an object. What must the one contract for 9.0 and later send so that clients from every pinned version accept it?

## Answer

The pinned files disagree, and no value satisfies all of them:

| Pinned version | `nextCursor` and `prevCursor` in the scan response |
|---|---|
| 9.0.1 (L-0001) | an object with two properties: a direction flag and a cursor kind |
| 9.2.4 (L-0002) | not declared |
| 9.5.1 (L-0003), 10.0.1 (L-0004), 10.2.20 (L-0005), 11.0.0 (L-0006), 11.1.0 (L-0007) | a string |

In all seven files the `cursor` request parameter is a string. `listSpaceTrash` appears from 9.5.1 and declares strings throughout.

The versions cannot be reconciled. A JSON value cannot be a string and an object at once. Leaving the fields out would satisfy every file, but a client from 9.5.1 on that pages with `nextCursor` would stop after one page.

The string shape wins, for three reasons:

- The client sends the cursor back as the `cursor` parameter, which is a string in every pinned version. Only a string can make that round trip.
- The 9.0.1 object holds no position, only a direction and a kind, so a client cannot resume a walk from it. It reads as the server's internal type leaking into that one file, not as a value a client was meant to use. 9.2.4 dropped it, and every later file declares a string.
- Five of the seven pinned versions, including every version from 9.5.1 on, declare a string.

So aerie keeps sending strings. The contract already said so; it now says it on purpose. Clients generated from the 9.0.1 file must not check these two fields strictly; following `_links.next` works for them as for any client. Serving the 9.0.1 shape when aerie presents a 9.0 version was considered and rejected: the presented version describes the server, and the server cannot know which file a client was generated from.

The lab's reading (that the contract disagrees with the 9.0.1 generated clients) is right. The contract's shape stands; what changes is that the spec now states the conflict and the choice.

The rest of the sweep: no other response or request body field in the 52 T1 operations has a type that differs between pinned versions. Query and path parameters do differ: `start`, `limit` and `version` are strings up to 9.5.1 and numbers from 10.0.1, and `status`, `ids` and `location` change from one value to a list. On the wire a query value is text either way, and one value is a list of one, so aerie accepts both forms and nothing changes.

Written in `behaviour/common.md` (Paging, cursor style); `scanContent` description and `ContentCursorPage` in the contract.
