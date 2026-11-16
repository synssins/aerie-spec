# Q-0024: What hide, delete and restore do, and who moderates

Raised by: the spec room, while specifying the owner's request for page comments.
Status: answered in pull request `phase1-features-1`.

## Where

Aerie's own API under `/api/v1`, not a compatible surface.

## Question

The owner asked for moderation by the space's Supervisors and Owners: hide, delete and restore. What does each do, how long are deleted comments kept, can authors' own deletions be restored, and do site administrators moderate?

## Answer

- **Hide**: the comment stays where it is, visible only to moderators and to its author (marked hidden). Reversible at any time.
- **Delete** by a moderator: the comment is visible only to moderators, and purged 30 days later.
- **Restore**: makes a hidden comment, or a moderator-deleted comment not yet purged, visible again.
- A comment its author deleted cannot be restored by anyone. A person who withdraws what they wrote must be able to rely on it being gone.
- Hiding or deleting a top-level comment leaves its replies; readers see a placeholder in its place.
- Only the space's Supervisors and Owners moderate. Being a site administrator is not enough on its own: an administrator who needs to moderate takes a role in the space the usual way, which is visible and audited. This follows the owner's wording ("moderated by the space's Supervisors and Owners").

Left to the owner: whether site administrators should also moderate every space without a role. This ruling says no.

Written in `behaviour/comments.md` (Who may do what, Moderation).
