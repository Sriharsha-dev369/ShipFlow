# ShipFlow — UX & User Flows

How the product behaves from the user's side: flows, screens, wireframes, empty states, and failure states. Deliberately about **UX over UI** — layout and behaviour, not colour and typography.

---

## 1. UX principles

1. **The core loop is one click from anywhere.** "See what needs attention → move something → capture new work." Anything on that path gets a keyboard shortcut and a permanent place in the chrome.
2. **No dead ends.** Every empty state names the next action. Every error says what to do, not just what failed.
3. **Optimistic where safe, honest where not.** Dragging a card updates instantly and rolls back on failure. Deleting something waits for the server.
4. **Never lose typed work.** A failed comment or issue submission keeps the text in the box.
5. **Keyboard-navigable throughout.** Tab order follows visual order; focus is always visible; no mouse-only affordances.

---

## 2. Screen inventory

| Screen | Route | Purpose |
|---|---|---|
| Sign up | `/register` | Create an account |
| Log in | `/login` | Authenticate |
| Accept invitation | `/invite/:token` | Join an organization |
| Onboarding / no org | `/welcome` | Create the first organization |
| Board (home) | `/orgs/:orgId/projects/:projectId/board` | **Center of gravity** — the daily loop |
| Issue list | `/orgs/:orgId/projects/:projectId/issues` | Filter, sort, search |
| Issue detail | `/orgs/:orgId/projects/:projectId/issues/:number` | Read, edit, comment, attach |
| Project list | `/orgs/:orgId/projects` | Switch projects |
| Members | `/orgs/:orgId/members` | Invite, change roles, remove |
| Activity | `/orgs/:orgId/activity` | Audit timeline |
| Org settings | `/orgs/:orgId/settings` | Rename, transfer ownership, delete |
| Project settings | `/orgs/:orgId/projects/:projectId/settings` | Rename, delete |
| Account settings | `/account` | Name, password |
| Notifications | Panel in header, not a page | Unread count + list |

---

## 3. Core flows

### 3.1 First-run — signup to first issue

```
/register
    │  email + password
    ▼
/welcome  ──────────────────► "Create your organization"
    │  org name                 (no orgs exist yet, so this is forced)
    ▼
Org created, user is OWNER
    │
    ▼
"Create your first project"   name + key (e.g. WEB)
    │
    ▼
/orgs/:id/projects/:id/board   ← empty board, 4 columns
    │
    ▼
"Create your first issue"     inline on the board, not a modal
```

**The whole path is five inputs**: email, password, org name, project name+key, issue title. Anything more and people leave.

### 3.2 The daily loop

```
Land on board (last-visited project remembered)
    │
    ├─► scan columns ──► drag card Todo → In Progress ──► optimistic update
    │
    ├─► click card ──► issue detail ──► comment / reassign / change priority
    │
    └─► press "c" (or click + Issue) ──► inline create ──► card appears in Todo
```

### 3.3 Inviting a teammate

```
/orgs/:id/members
    │  "Invite" → email + role
    ▼
Invitation created
    │
    ├─ Slice 2: link shown on screen with a "Copy link" button
    └─ Slice 6: link also emailed automatically
    │
    ▼
Invitee opens /invite/:token
    │
    ├─ not signed in ──► register or log in ──► returns to the invite
    └─ signed in ─────► "Join <Org> as MEMBER?"  [Accept] [Decline]
    │
    ▼
Membership created ──► lands on that org's board
```

Accepting is always an explicit click, even for an existing account. Nobody gets added to an organization they didn't agree to join.

### 3.4 Assignment → notification

```
User A assigns issue WEB-42 to User B
        │
        ▼
User B's next page load: bell shows "1"
        │
        ▼
Opens panel ──► "A assigned you WEB-42" ──► click ──► issue detail, marked read
```

---

## 4. Wireframes

### 4.1 Board — the home screen

```
┌────────────────────────────────────────────────────────────────────────┐
│ ShipFlow   [Acme Corp ▾]  [Web App ▾]        🔍  🔔3   [HS ▾]          │
├────────────────────────────────────────────────────────────────────────┤
│  Board   Issues   Members   Activity                       [+ Issue]   │
├────────────────────────────────────────────────────────────────────────┤
│  Assignee: [Anyone ▾]   Priority: [Any ▾]                              │
├──────────────┬──────────────┬──────────────┬──────────────────────────┤
│ TODO      4  │ IN PROGRESS 2│ IN REVIEW  1 │ DONE                  7  │
├──────────────┼──────────────┼──────────────┼──────────────────────────┤
│ ┌──────────┐ │ ┌──────────┐ │ ┌──────────┐ │ ┌──────────┐             │
│ │ WEB-42   │ │ │ WEB-38   │ │ │ WEB-31   │ │ │ WEB-29   │             │
│ │ Fix login│ │ │ Add board│ │ │ Rate lim │ │ │ Setup CI │             │
│ │ redirect │ │ │ drag+drop│ │ │ on login │ │ │          │             │
│ │ 🔴 High  │ │ │ 🟡 Med   │ │ │ 🔴 High  │ │ │ 🟢 Low   │             │
│ │      (HS)│ │ │      (RK)│ │ │      (HS)│ │ │      (PM)│             │
│ └──────────┘ │ └──────────┘ │ └──────────┘ │ └──────────┘             │
│ ┌──────────┐ │ ┌──────────┐ │              │ ┌──────────┐             │
│ │ WEB-41   │ │ │ WEB-40   │ │              │ │ WEB-27   │             │
│ │ ...      │ │ │ ...      │ │              │ │ ...      │             │
│ └──────────┘ │ └──────────┘ │              │ └──────────┘             │
└──────────────┴──────────────┴──────────────┴──────────────────────────┘
```

A card shows only: number, title, priority, assignee avatar. Anything more and you can't scan a column.

### 4.2 Issue detail

```
┌────────────────────────────────────────────────────────────────────────┐
│ ← Board                                                        [Delete]│
├────────────────────────────────────────────────────────────────────────┤
│ WEB-42                                                                 │
│ ┌────────────────────────────────────────┐  ┌────────────────────────┐ │
│ │ Fix login redirect loop                │  │ Status   [In Progress▾]│ │
│ │ (click to edit inline)                 │  │ Assignee [Harsha    ▾] │ │
│ ├────────────────────────────────────────┤  │ Priority [High      ▾] │ │
│ │ After logging in, users bounce between │  │                        │ │
│ │ /login and /board. Repros on Safari.   │  │ Reporter  Rahul K      │ │
│ │                                        │  │ Created   2 days ago   │ │
│ │ (click to edit)                        │  │ Updated   4 hours ago  │ │
│ └────────────────────────────────────────┘  └────────────────────────┘ │
│                                                                        │
│ ATTACHMENTS                                              [+ Attach]    │
│  📎 safari-console.png   240 KB · Rahul K · yesterday          [×]     │
│                                                                        │
│ COMMENTS (2)                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ (RK) Rahul K · 2 days ago                                        │  │
│  │ Only happens when the refresh cookie is missing.                 │  │
│  ├──────────────────────────────────────────────────────────────────┤  │
│  │ (HS) Harsha · 3 hours ago · edited              [Edit] [Delete]  │  │
│  │ Confirmed — SameSite issue. Fixing.                              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Write a comment…                                       [Comment] │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

Status, assignee, and priority are editable **in place from the sidebar** — no edit mode, no save button. Changing status here and dragging on the board hit the same endpoint.

### 4.3 Members

```
┌────────────────────────────────────────────────────────────────────────┐
│ Members — Acme Corp                                    [+ Invite]      │
├────────────────────────────────────────────────────────────────────────┤
│ MEMBERS (3)                                                            │
│  (HS) Harsha S      harsha@…       OWNER                               │
│  (RK) Rahul K       rahul@…        [ADMIN  ▾]              [Remove]    │
│  (PM) Priya M       priya@…        [MEMBER ▾]              [Remove]    │
│                                                                        │
│ PENDING INVITATIONS (1)                                                │
│  dev@example.com    MEMBER   expires in 6 days   [Copy link] [Revoke]  │
└────────────────────────────────────────────────────────────────────────┘
```

The OWNER row has no role dropdown and no Remove button — the invariant is expressed in the UI, not just rejected by the API.

### 4.4 Invite modal (Slice 2 — link only)

```
┌────────────────────────────────────────────┐
│ Invite to Acme Corp                    [×] │
├────────────────────────────────────────────┤
│ Email    [ dev@example.com            ]    │
│ Role     [ MEMBER ▾ ]                      │
│                              [Cancel][Send]│
├────────────────────────────────────────────┤
│ ✓ Invitation created                       │
│ Share this link — expires in 7 days:       │
│ ┌────────────────────────────────────────┐ │
│ │ https://shipflow.app/invite/a3f9…  📋  │ │
│ └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

From Slice 6 the same modal says "We've emailed them" and keeps the copy-link as a fallback — the UI barely changes, which is the point of splitting the slice that way.

---

## 5. Empty states

Every one names the next action. None of them is a shrug.

| Where | Shows |
|---|---|
| No organizations | "You're not in an organization yet. **Create one** — or ask a teammate for an invite link." |
| No projects | "No projects yet. **Create your first project** to start tracking work." |
| Empty board | "Nothing here yet. **Create your first issue** — press `c` or click + Issue." |
| Empty column | Dashed drop zone: "Drag issues here" |
| No comments | "No comments yet. Start the conversation." |
| No notifications | "You're all caught up." |
| No search results | "No issues match '<query>'. **Clear filters** or try different words." |
| No activity | "Nothing has happened yet. Activity will appear as your team works." |

---

## 6. Failure states

| Failure | What the user sees |
|---|---|
| Wrong login credentials | "Email or password is incorrect." — never which one was wrong |
| Rate-limited login | "Too many attempts. Try again in a few minutes." |
| Session expired mid-action | Silent token refresh; only if that fails → "Your session expired. **Sign in again**", with typed content preserved |
| Drag-and-drop fails | Card animates back to its original column + toast: "Couldn't move WEB-42. Try again." |
| Comment submission fails | Text stays in the box, inline error, retry button |
| File too large | Rejected **before upload starts**: "Files must be under 10 MB. This one is 14.2 MB." |
| Disallowed file type | "That file type isn't supported. Allowed: images, PDF, text, documents, zip." |
| Expired invitation | "This invitation expired. Ask <org> for a new one." |
| Already-accepted invitation | "You're already a member of <org>." → link to its board |
| Access to another org's resource | 404, not 403 — never confirm that a resource exists in an org you can't see |
| Server error | "Something went wrong on our end." + retry. Never a stack trace |

---

## 7. Mobile behaviour

| Desktop | Mobile |
|---|---|
| 4-column board, drag to change status | Single status-filtered list; status changes via a dropdown on the card |
| Issue detail two-column | Single column, sidebar fields collapse above the description |
| Persistent left nav | Hamburger menu |
| Hover actions (edit/delete) | Always-visible action menu |

Touch drag-and-drop is not implemented. The mobile board is a filtered list by design, not a degraded desktop board.

---

## 8. Keyboard shortcuts

| Key | Action |
|---|---|
| `c` | Create issue (from board or issue list) |
| `/` | Focus search |
| `Esc` | Close modal / cancel inline edit |
| `g` then `b` | Go to board |
| `g` then `i` | Go to issue list |

Everything reachable by mouse is reachable by keyboard, whether or not it has a shortcut.
