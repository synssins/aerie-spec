# Q-0004: Spaces without a home page

Raised by: the compatibility lab, first run against a real aerie image.
Status: answered in pull request `phase1-rulings-1`.

## Where

Data Center compatible surface: the `homepage` field and its expansions on spaces (`GET /rest/api/space`, `GET /rest/api/space/{spaceKey}`), and `homepage` in the create and update bodies.

## Question

Aerie spaces have no home page. What does the compatible surface return for `homepage` and its expansions?

## Answer

- `homepage` is never returned and is not listed in `_expandable`. `expand=homepage`, and deeper paths under it, are ignored like any unknown expansion.
- Creating a space creates no page. `homepage` in a create or update body is ignored.
- To walk a space from its top, clients list `GET /rest/api/space/{spaceKey}/content/page?depth=root`.

Written in `behaviour/spaces.md` (No home page, Creating, Updating), with `Space`, `SpaceWrite`, `createSpace` and `updateSpace` in `contracts/datacenter/openapi.yaml`.
