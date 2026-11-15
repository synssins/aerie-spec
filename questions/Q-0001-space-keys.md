# Q-0001: Space keys: case on the compatible surface

Raised by: the compatibility lab, first run against a real aerie image.
Status: answered in pull request `phase1-rulings-1`.

## Where

Data Center compatible surface: every operation that takes or returns a space key. That includes `GET/PUT/DELETE /rest/api/space/{spaceKey}`, `POST /rest/api/space`, the `spaceKey` query filters and CQL `space`.

## Question

Clients of the compatible surface send and expect upper-case or mixed-case space keys, such as `LAB`. Aerie's own keys are 2 to 32 characters of lower-case letters, digits and hyphens. How are keys accepted, stored, compared and returned on the compatible surface?

## Answer

- Returned: the aerie key in upper case (`lab` is returned as `LAB`).
- Accepted: any case. Aerie lower-cases a key before it looks it up.
- Stored: a key sent to create a space is lower-cased, then checked against aerie's own key rules. A key that breaks them gives `400`.
- Compared: always without regard to case.

Written in `behaviour/spaces.md` (Space keys, Creating, Updating), with the key description in `contracts/datacenter/openapi.yaml` (`Space`, `SpaceWrite`, `spaceKeyPath`).

Still open: aerie keys may hold hyphens. A client that checks the characters of a key may reject one such as `TEAM-A`. This is listed for the lab in `spaces.md`.
