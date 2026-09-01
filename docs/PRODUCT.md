# ShipFlow — Product Specification

The user-facing definition of ShipFlow: who it's for, what a person can do with it, and what it refuses to do. Written for the dev team, not for agents. Vocabulary comes from [`../CONTEXT.md`](../CONTEXT.md).

---

## 1. What ShipFlow is

A multi-tenant project and issue tracker for small software teams. A person signs up, creates an organization, brings teammates in, and tracks work as issues moving across a board.

## 2. Who it's for

**Anchor persona: a tech lead on a 6-person product team.**

They currently track work in GitHub Issues, and the pain is specific: GitHub Issues is *repo-scoped*, so work spanning two repos — or work that isn't code at all (design, infra, a customer follow-up) — has nowhere to live. They've tried Trello (no issue semantics: assignee and status carry no real meaning, no history of who changed what) and rejected Jira (fifteen minutes of configuration before you can file one ticket).

**The wedge**: cross-project work tracking with real roles and real history, without Jira's setup tax.

Use this persona as the decision rule for scope arguments: *would a 6-person team's lead actually need this?* If the answer needs a paragraph, the answer is no.

## 3. The core loop

> **See what needs my attention, move one piece of work forward, capture new work before it's lost.**

The consequence, which drives every UX decision: **home is not a projects list.** A projects index is a filing cabinet, and nobody opens a filing cabinet daily. Home is work-centric — the board, or a cross-project view of what's assigned to you.

## 4. Access model

- **Registration is open.** Anyone can sign up with an email and password.
- **Any signed-up user can create an organization**, and becomes its OWNER.
- **Invitation is the only path into an existing organization.** No email-domain auto-join (needs domain verification to be safe), no public join links (a leaked link is a tenant breach with no revocation story).
- **Nothing is visible without membership.** No public projects, no guest role, no shareable read-only links. One access rule — "are you a member of this organization?" — applied uniformly.

The accepted cost of that last one: there is no "share this issue with a client" flow, and adding one later means introducing a second access path that every query and guard must then account for.

## 5. Scale assumptions

| Dimension | Expected | Consequence |
|---|---|---|
| Members per organization | 3–15 | Member lists need no pagination — fetch whole |
| Projects per organization | 1–10 | Project lists need no pagination |
| Issues per project | 100–500 over a year | **Pagination and search are day-one requirements** |
| Organizations per user | 1–3 | An org switcher is needed, but it can be a simple dropdown |

Board column virtualization is explicitly **not** needed: a column with 500 cards is a product failure (the user should be filtering), not a rendering problem.

## 6. Platform expectations

- **Desktop-first.** The board with drag-and-drop is a desktop experience.
- **Mobile responsive for reading and status changes.** On small screens the board degrades to a status-filtered list where status changes via a menu, not a drag. Touch drag-and-drop is deliberately out of scope.
- **No offline support.** Requires a connection.
- **Evergreen browsers only.** No IE, no legacy Safari.
- **Accessibility bar we hold ourselves to**: keyboard-navigable throughout, semantic HTML, real form labels, visible focus states. Cheap when done from the start, miserable to retrofit.

---

## 7. Capabilities — what a user can actually do

Exhaustive. If it isn't listed here, it isn't in the product.

### 7.1 Account

- Register with email + password
- Log in; log out
- Stay signed in across page reloads and browser restarts
- Change display name
- Change password — requires the current password, and signs out all other sessions
- See which organizations they belong to

*Not in scope:* forgotten-password reset, email verification enforcement, account deletion, social login.

### 7.2 Organization

| Action | Who |
|---|---|
| Create an organization (creator becomes OWNER) | Any signed-in user |
| Switch between organizations they belong to | Any member |
| View the organization's member list | Any member |
| Rename the organization | OWNER, ADMIN |
| Invite someone by email, with a role | OWNER, ADMIN |
| Revoke a pending invitation | OWNER, ADMIN |
| Accept an invitation via its link | The invited person |
| Change another member's role | OWNER, ADMIN (never the OWNER's role) |
| Remove a member | OWNER, ADMIN (never the OWNER) |
| Leave the organization | Any member except the OWNER |
| Transfer ownership to another member | OWNER only |
| Delete the organization | OWNER only |

### 7.3 Project

| Action | Who |
|---|---|
| Create a project (name + short key, e.g. `WEB`) | Any member |
| List projects in the organization | Any member |
| Open a project | Any member |
| Rename a project | OWNER, ADMIN |
| Delete a project | OWNER, ADMIN |

Every member can see every project in their organization. There is no per-project membership.

### 7.4 Issue

| Action | Who |
|---|---|
| Create an issue (title required; description, priority, assignee optional) | Any member |
| Read an issue, with its auto-assigned number (e.g. `WEB-142`) | Any member |
| Edit title and description | Any member |
| Change status (Todo → In Progress → In Review → Done, any direction) | Any member |
| Change priority (Low / Medium / High / Urgent) | Any member |
| Assign to, or unassign from, any organization member | Any member |
| Delete an issue | Any member |
| Filter the issue list by status, assignee, priority | Any member |
| Sort by created, updated, or priority | Any member |
| Search issues by text in title/description | Any member |
| View the board grouped by status, and drag a card to change status | Any member (desktop) |

Issue permissions are intentionally flat: a MEMBER who can edit and reassign any issue but not delete one would be an inconsistent carve-out.

### 7.5 Comment

| Action | Who |
|---|---|
| Comment on an issue | Any member |
| Read an issue's comments, oldest first | Any member |
| Edit own comment (displays as "edited") | The author |
| Delete own comment | The author |
| Delete someone else's comment | OWNER, ADMIN (delete only — never edit) |

Comments are flat. There are no threaded replies.

### 7.6 File

| Action | Who |
|---|---|
| Attach a file to an issue | Any member |
| See a file's name, size, type, uploader, and upload date | Any member |
| Download an attachment | Any member |
| Delete an attachment | The uploader, OWNER, ADMIN |

### 7.7 Notification

| Action | Who |
|---|---|
| Receive an in-app notification when assigned an issue | The assignee |
| Receive an in-app notification when someone comments on an issue they're assigned to | The assignee |
| See an unread count | The recipient |
| Mark one, or all, as read | The recipient |
| Click through to the issue that triggered it | The recipient |

Notifications belong to a person, not an organization. Only the recipient can ever see them.

### 7.8 Activity / audit

| Action | Who |
|---|---|
| View the organization's activity timeline, newest first | Any member |
| See who did what, to which resource, and when | Any member |

Audit history is a shared team artifact here, not a restricted admin tool — which means nothing may be recorded in it that a regular MEMBER shouldn't see.

### 7.9 What the system does without being asked

- Assigns each issue a per-project sequential number
- Writes an audit entry for every tracked action (§7.8)
- Creates notifications on assignment and on comments
- Expires unaccepted invitations after 7 days
- Emails the invitation link via a background worker
- Rotates refresh tokens on every use
- Rate-limits abusive request patterns

---

## 8. Guardrails

Limits and invariants the product enforces. These are product rules, not implementation details — the API and UI must both respect them.

| Guardrail | Value |
|---|---|
| File size | ≤ 10 MB per file |
| File types | Allowlist: images, PDF, plain text/markdown, common office docs, zip |
| Invitation lifetime | 7 days |
| Pending invitations | One per (organization, email) — re-inviting extends, never duplicates |
| Organization OWNER | Exactly one, always. Cannot leave or be removed without transferring first |
| Project key | 2–6 characters, uppercase alphanumeric, unique within the organization |
| Issue title | 1–200 characters |
| Issue description | ≤ 10,000 characters |
| Comment body | 1–5,000 characters |
| Organization / project name | 1–100 characters |
| Login attempts | 5 per 15 minutes, per IP |
| Authenticated requests | 100 per minute, per user |

**Invariants that must never break:**
- A user can only ever read or write data belonging to an organization they are a member of.
- An organization always has exactly one OWNER.
- A deleted organization, project, or issue disappears from the product but remains referenceable by its audit history.

---

## 9. Non-goals

Deliberate cuts, each with a reason. When someone asks "why doesn't it do X?", the answer is here, not "we ran out of time."

| Not building | Why |
|---|---|
| Real-time collaborative board | Refetch-on-focus covers the actual need for a small team; a websocket gateway is a separate architectural component |
| Configurable per-project workflows | Turns every status read into a join and adds a workflow-management CRUD surface, for a product that isn't a workflow platform |
| Per-project membership | Adds a second authorization layer to every issue operation, for teams small enough that the org is already the trust boundary |
| Multiple assignees per issue | "Who owns this" should have one answer |
| Public sharing / guest access | Introduces a second access path that every query must account for, forever |
| OAuth / SSO | Bounded, addable later; not foundational |
| Email notifications beyond invitations | The invite email proves the async pipeline; generalizing it before there's a second real need is speculative |
| Password reset | Nice-to-have; needs the email pipeline, which arrives late |
| Mobile app, analytics, billing, AI features | v2 directions, not requirements for this product |
