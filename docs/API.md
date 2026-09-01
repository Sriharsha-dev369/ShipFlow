# ShipFlow — API Design

REST over HTTPS, JSON in and out. Documented with OpenAPI/Swagger generated from NestJS decorators.

---

## 1. Conventions

**Resource naming.** Plural nouns, kebab-case, nested to express ownership:
`/organizations/:orgId/projects/:projectId/issues/:issueId`

**The organization is always in the path.** Never a header, never a token claim. Every org-scoped route is `/organizations/:orgId/...`, and one guard checks membership in `:orgId` for all of them — it is structurally impossible to add an org-scoped endpoint that forgets the check. See ADR-0007.

**Methods and status codes.**

| Method | Use | Success |
|---|---|---|
| GET | Read | 200 |
| POST | Create, or a non-CRUD action | 201 created / 200 action |
| PATCH | Partial update | 200 |
| DELETE | Remove | 204 |

| Failure | Code |
|---|---|
| Validation failed | 422 |
| Malformed request | 400 |
| Not authenticated | 401 |
| Authenticated but not permitted | 403 |
| Resource absent — **or in an org you can't see** | 404 |
| Conflict (duplicate key, invariant violation) | 409 |
| Rate limited | 429 |

**404-not-403 for cross-tenant access is deliberate.** Returning 403 for another organization's issue confirms that the issue exists — a membership oracle. Cross-tenant reads look identical to nonexistent resources.

**Actions that aren't CRUD** get a verb sub-resource rather than a magic field:
`POST /organizations/:orgId/transfer-ownership`, `POST /invitations/:token/accept`, `POST /notifications/read-all`.

**Error envelope** — identical shape everywhere, produced by one global exception filter:

```json
{
  "statusCode": 422,
  "error": "Unprocessable Entity",
  "message": "Validation failed",
  "details": [
    { "field": "title", "message": "must be between 1 and 200 characters" }
  ],
  "requestId": "01HQ8X…"
}
```

`details` is present only for validation failures. `requestId` appears on every error and is the same id in the structured logs — the thing that makes a user's bug report traceable.

**Pagination is cursor-based**, on `(created_at, id)`:

```
GET /…/issues?limit=50&cursor=eyJjIjoiMjAy…

{ "data": [...], "nextCursor": "eyJjIjoi…", "hasMore": true }
```

Offset pagination drifts when rows are inserted or deleted mid-scroll — a user paging through issues while a teammate files new ones sees duplicates and gaps. Cursor pagination doesn't, and it stays fast at any depth.

**Filtering and sorting** are query params, allowlisted server-side — never a raw column name from the client:
`?status=TODO,IN_PROGRESS&assigneeId=…&priority=HIGH&sort=-updatedAt&q=login`

**Validation** happens at the edge via class-validator DTOs with `forbidNonWhitelisted`, so unknown fields are rejected rather than silently ignored.

**Versioning**: none. There are no external consumers; the web app ships with the API. Introducing `/v1/` before anyone depends on it is ceremony.

**Idempotency.** A network retry double-submitting a `POST` is the real risk this addresses — not a theoretical one, since TanStack Query and mobile networks both retry. Where it's already handled structurally, no extra work is needed: registration and invitation-accept are protected by unique constraints on email; invitation creation is protected by the pending-invite unique constraint. Where it isn't — `POST` on issues and comments — a duplicate is low-consequence (the user sees two cards, deletes one) and not worth an `Idempotency-Key` header and a dedup-table for this product's scale. **This is a deliberate scope cut, not an oversight**: revisit it if a real duplicate-submission bug ever shows up, rather than building the mechanism speculatively.

---

## 2. Endpoint map

Grouped by slice, so it doubles as a build order. Not exhaustive-by-design — endpoints exist because a screen in [`UX.md`](./UX.md) needs them.

### Auth — Slice 1
```
POST   /auth/register              → 201, sets refresh cookie, returns access token
POST   /auth/login                 → 200, sets refresh cookie, returns access token
POST   /auth/refresh               → 200, rotates refresh cookie, new access token
POST   /auth/logout                → 204, revokes refresh token
GET    /auth/me                    → 200, current user + their organizations
```

### Account — Slice 1
```
PATCH  /account                    → 200  (name)
POST   /account/change-password    → 204  (current + new; revokes all other refresh tokens)
```

### Organizations & membership — Slice 2
```
POST   /organizations                             → 201, creator becomes OWNER
GET    /organizations                             → 200, orgs the caller belongs to
GET    /organizations/:orgId                      → 200
PATCH  /organizations/:orgId                      → 200  (rename)          [ADMIN+]
DELETE /organizations/:orgId                      → 204  (soft)            [OWNER]
POST   /organizations/:orgId/transfer-ownership   → 204                    [OWNER]

GET    /organizations/:orgId/members              → 200
PATCH  /organizations/:orgId/members/:memberId    → 200  (change role)     [ADMIN+]
DELETE /organizations/:orgId/members/:memberId    → 204  (remove)          [ADMIN+]
DELETE /organizations/:orgId/members/me           → 204  (leave)           [not OWNER]
```

### Invitations — Slice 2 (link) / Slice 6 (email)
```
POST   /organizations/:orgId/invitations          → 201, returns the invite link  [ADMIN+]
GET    /organizations/:orgId/invitations          → 200, pending only             [ADMIN+]
DELETE /organizations/:orgId/invitations/:id      → 204  (revoke, hard delete)    [ADMIN+]
GET    /invitations/:token                        → 200, org name + role (public, pre-auth preview)
POST   /invitations/:token/accept                 → 201, creates membership       [authenticated]
```

`/invitations/:token` is deliberately **not** org-nested: the person accepting isn't a member yet, so the org guard cannot apply. It's the one route outside the uniform pattern, and it's the one route that gets extra scrutiny in review.

### Projects — Slice 3
```
POST   /organizations/:orgId/projects             → 201
GET    /organizations/:orgId/projects             → 200
GET    /organizations/:orgId/projects/:projectId  → 200
PATCH  /organizations/:orgId/projects/:projectId  → 200  (rename)   [ADMIN+]
DELETE /organizations/:orgId/projects/:projectId  → 204  (soft)     [ADMIN+]
```

### Issues — Slice 4
```
POST   /organizations/:orgId/projects/:projectId/issues            → 201
GET    /organizations/:orgId/projects/:projectId/issues            → 200  (filter/sort/paginate)
GET    /organizations/:orgId/projects/:projectId/issues/:number    → 200  (by WEB-42 number, not uuid)
PATCH  /organizations/:orgId/projects/:projectId/issues/:number    → 200  (title/desc/status/priority/assignee)
DELETE /organizations/:orgId/projects/:projectId/issues/:number    → 204  (soft)
GET    /organizations/:orgId/issues?assigneeId=me                  → 200  (cross-project "my issues")
```

Issues are addressed by their human-readable `number` within a project, not their uuid — the URL matches what people say out loud ("look at WEB-42").

A single `PATCH` handles status changes, so dragging a card on the board and changing status in the sidebar are the same request.

### Comments — Slice 5
```
POST   /…/issues/:number/comments        → 201
GET    /…/issues/:number/comments        → 200  (chronological, paginated)
PATCH  /…/comments/:commentId            → 200  (author only, sets edited_at)
DELETE /…/comments/:commentId            → 204  (author, or ADMIN+)
```

### Notifications — Slice 6
```
GET    /notifications                    → 200  (caller's own, across all orgs)
GET    /notifications/unread-count       → 200
POST   /notifications/:id/read           → 204
POST   /notifications/read-all           → 204
```

Not org-nested — a notification belongs to a person, and the bell shows work from every org they're in.

### Files — Slice 7
```
POST   /…/issues/:number/files/upload-url   → 201  { fileId, uploadUrl }  — authorised before the URL is minted
POST   /…/files/:fileId/complete            → 200  — client confirms the upload finished
GET    /…/issues/:number/files              → 200
GET    /…/files/:fileId/download-url        → 200  { downloadUrl }  — short-lived signed GET
DELETE /…/files/:fileId                     → 204  (uploader, or ADMIN+)
```

File bytes never pass through the API — only signed URLs do.

### Activity — Slice 8
```
GET    /organizations/:orgId/audit-log      → 200  (paginated, newest first)
```

### Health — Slice 0
```
GET    /health          → 200 always, if the process is alive  (liveness)
GET    /health/ready    → 200 / 503 based on Postgres + Redis  (readiness)
```

Split deliberately: a liveness probe that fails when the database blips causes the platform to kill a healthy process and make an outage worse.
