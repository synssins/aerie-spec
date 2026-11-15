# Q-0019: What `args` holds in message objects

Raised by: the compatibility lab, black-box run with clients generated from each pinned published file.
Status: answered in pull request `phase1-rulings-4`.

## Where

Data Center compatible surface: the `Message` schema. It appears in `GET /rest/api/longtask` (`listLongTasks`) and `GET /rest/api/longtask/{id}` (`getLongTask`), as `name` and in `messages`, and in the `data.errors[].message` of every error body (`RestError`).

## Question

Aerie answers the long-task reads with a `name` message whose `args` is a list of strings, for example `["DS16"]`. Clients generated in Python from the pinned files reject it: each item must be a dictionary. Java and TypeScript clients accept it. Our contract left the items untyped. What are `args` items on the compatible surface, so clients generated from every pinned version accept the answer?

## Answer

`args` is always an empty list.

All seven pinned files (L-0001 to L-0007) declare `args` the same way: a list whose items are objects, with no fields named. A string is not an object, so `["DS16"]` fails strict clients. An object item would pass, but its fields would have to be aerie's invention, because no pinned file names any. An empty list is valid against every pinned file and against strict and loose clients alike.

So aerie puts the values a message is about into `translation`, the full text with every value filled in (`Deleting space DS16`). `key` stays a stable identifier for the kind of message. The contract now types the items as objects and says the list is empty (`maxItems: 0`). The behaviour is in `common.md` under Message objects.

A client that read the space key out of `args[0]` loses it. No pinned file says what `args` holds, so no client can have relied on its content from the published record; an empty list is valid for the Java and TypeScript clients that accepted the strings.

## Sweep

Every schema reached from the 56 operations in the contract (the 52 T1 operations under `/rest/api` and the 4 OAuth 2.0 operations) was checked for an untyped value, an untyped array item, or an object with no fields, and compared with the pinned files at the same place. Three were found:

| Place | Pinned files | Result |
|---|---|---|
| `Message.args` items | object, in all seven | Tightened, as above |
| `Content.container` | an empty schema (any value), in all seven | Already matches: ours is an object, which every client accepts |
| `Content.restrictions` | an object with no fields named, in all seven | Already matches |

The 4 OAuth 2.0 operations are not in the pinned files and have no loose schemas. No other schema needed tightening.

Written in `behaviour/common.md` (Errors, Message objects, Long tasks); `Message` in the contract.
