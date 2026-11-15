# Q-0002: Page hierarchy: child pages on top of folders

Raised by: the compatibility lab, first run against a real aerie image.
Status: answered in pull request `phase1-rulings-1`.

## Where

Data Center compatible surface: `GET /rest/api/content/{id}/child` and `/child/page`, `/descendant` and `/descendant/page/count`, `ancestors` on content, `POST /rest/api/content` and `PUT /rest/api/content/{id}` with `ancestors`, and CQL `parent` and `ancestor`.

## Question

On the compatible surface, any page can have child pages. In aerie, only folders hold other nodes; a page holds none. How do child pages, ancestors and descendants map onto aerie's tree?

## Answer

- Folders and pages both appear as `type: page`. A folder's body is its own content, and is empty when the folder has no text.
- Children, descendants and ancestors follow aerie's tree. A page (as opposed to a folder) has no children.
- A page that gains a child through the compatible surface becomes a folder in place. It keeps its id, title, body, versions, labels and attachments. A folder never turns back into a page.

Written in `behaviour/content.md` (How the page tree maps onto aerie's tree, Creating, Updating), with the `createContent` description in `contracts/datacenter/openapi.yaml`.

For the build room: a folder must keep and show its own body, because content created through the compatible surface may give a folder text. Aerie's data already stores a body for folders; the web app needs to show and edit it.
