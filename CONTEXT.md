# ShipFlow

A multi-tenant project and issue tracker for small software teams: organizations, projects, issues, and the people who work on them.

## Language

**Organization**:
The tenant boundary. All data (projects, issues, files, invitations) belongs to exactly one Organization. A User may belong to many Organizations.
_Avoid_: Workspace, Team, Tenant (use Organization consistently)

**User**:
A person with a ShipFlow account, identified by email. A User's relationship to any given Organization is represented by an OrganizationMember, not by a direct field on User.

**OrganizationMember**:
The join between a User and an Organization, carrying that User's Role within that Organization. This is where org-scoped identity lives, not on the User itself.

**Role**:
The permission level an OrganizationMember holds within one Organization (OWNER, ADMIN, MEMBER). Scoped per-organization, not global to the User — a User can be OWNER of one Organization and MEMBER of another. Exactly one OWNER exists per Organization at a time; ownership moves via an explicit transfer action, never by simply adding a second OWNER.

**Project**:
A container for Issues, belonging to exactly one Organization. Every OrganizationMember has access to every Project in their Organization (no separate per-project membership in MVP).
_Avoid_: Board, Workspace

**Issue**:
A unit of trackable work belonging to exactly one Project. Has a Status drawn from a fixed, global set (not configurable per-project), and at most one assignee.
_Avoid_: Task, Ticket (pick Issue and use it consistently)

**IssueStatus**:
The fixed, global lifecycle state of an Issue (e.g. Todo, In Progress, In Review, Done). Not a per-project customizable workflow.

**Invitation**:
A pending offer for a specific email address to join a specific Organization with a specific Role, created by an existing OrganizationMember. Expires after a fixed window if not accepted. Accepting always requires an explicit action, even when the invited email already has a ShipFlow account.

**Notification**:
An in-app record telling a User something happened that concerns them (e.g. assigned to an Issue). Scoped to the receiving User, not the Organization.

**Comment**:
A flat (non-threaded), timestamped remark on an Issue. Its author may edit or delete it; an ADMIN or OWNER may also delete (but not edit) another member's Comment.

**File**:
An uploaded attachment on an Issue. ShipFlow stores only its metadata (name, size, MIME type, uploader, storage key); the object itself lives in object storage, never in Postgres.

**AuditLog**:
An immutable record of a significant action taken within an Organization (who, what, on which resource, when), visible to every OrganizationMember as a shared activity history. Distinct from application/operational logs, which are about system behavior, not organizational history.
