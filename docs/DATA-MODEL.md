# ShipFlow — Data Model

PostgreSQL is the source of truth. Prisma is the client and migration tool. Entity names follow [`../CONTEXT.md`](../CONTEXT.md).

Prisma models are PascalCase singular; tables and columns map to `snake_case` via `@@map`/`@map`, so the raw SQL we do write (the search index, the issue counter) reads like normal Postgres instead of quoted `"createdAt"` identifiers.

---

## 1. Entities

### User
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| email | citext | **unique** — case-insensitive so `A@x.com` and `a@x.com` are one account |
| password_hash | text | argon2id |
| name | text | display name |
| created_at / updated_at | timestamptz | |

### RefreshToken
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK → User | cascade delete |
| token_hash | text | **unique** — the raw token is never stored |
| expires_at | timestamptz | |
| revoked_at | timestamptz nullable | set on rotation, logout, or password change |
| created_at | timestamptz | |

Exists so refresh tokens can be *rotated and revoked*. Without it, "changing your password signs out other sessions" is unimplementable.

### Organization
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| name | text | 1–100 chars |
| created_at / updated_at | timestamptz | |
| deleted_at | timestamptz nullable | soft delete |

### OrganizationMember
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK → User | |
| organization_id | uuid FK → Organization | |
| role | enum `OWNER \| ADMIN \| MEMBER` | |
| created_at | timestamptz | |

Constraints: `unique (user_id, organization_id)` · `unique (organization_id) where role = 'OWNER'`

The second is a **partial unique index** enforcing "exactly one OWNER" in the database, not just in service code. Prisma's schema DSL can't express it — it's added as raw SQL in the migration.

### Invitation
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| organization_id | uuid FK → Organization | |
| email | citext | who it's for |
| role | enum | role granted on acceptance |
| token_hash | text | **unique** — raw token only ever lives in the link |
| invited_by_user_id | uuid FK → User | |
| expires_at | timestamptz | created_at + 7 days |
| accepted_at | timestamptz nullable | |
| created_at | timestamptz | |

Constraint: `unique (organization_id, email) where accepted_at is null` — one pending invitation per person per org. Re-inviting extends the existing row; it never creates a duplicate. Also a partial index, also raw SQL.

Revoked invitations are hard-deleted: nothing references them afterward.

### Project
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| organization_id | uuid FK → Organization | |
| name | text | 1–100 chars |
| key | text | 2–6 chars, uppercase alphanumeric — the `WEB` in `WEB-42` |
| last_issue_number | int default 0 | per-project issue counter (see §4) |
| created_at / updated_at | timestamptz | |
| deleted_at | timestamptz nullable | |

Constraint: `unique (organization_id, key)`

### Issue
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| project_id | uuid FK → Project | |
| number | int | per-project sequential, forms `WEB-42` |
| title | text | 1–200 chars |
| description | text nullable | ≤ 10,000 chars |
| status | enum `TODO \| IN_PROGRESS \| IN_REVIEW \| DONE` | default TODO |
| priority | enum `LOW \| MEDIUM \| HIGH \| URGENT` | default MEDIUM |
| assignee_id | uuid FK → User nullable | at most one |
| reporter_id | uuid FK → User | who created it |
| search_vector | tsvector generated | from title + description (Slice 9) |
| created_at / updated_at | timestamptz | |
| deleted_at | timestamptz nullable | |

Constraint: `unique (project_id, number)` — the safety net behind the counter in §4.

Any status can move to any other status. There are no illegal transitions; a team that wants to drag Todo straight to Done is allowed to.

### Comment
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| issue_id | uuid FK → Issue | cascade |
| author_id | uuid FK → User | |
| body | text | 1–5,000 chars |
| edited_at | timestamptz nullable | drives the "(edited)" marker |
| created_at | timestamptz | |

**Hard-deleted.** No audit event references a comment, and nothing points at one after removal.

### File
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| issue_id | uuid FK → Issue | |
| uploader_id | uuid FK → User | |
| file_name | text | original name, display only |
| mime_type | text | validated against the allowlist |
| size_bytes | bigint | ≤ 10 MB |
| storage_key | text | object key in R2 — never user-controlled |
| created_at | timestamptz | |
| deleted_at | timestamptz nullable | |

The object itself never touches Postgres.

### Notification
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK → User | the recipient |
| type | enum `ISSUE_ASSIGNED \| ISSUE_COMMENTED` | |
| payload | jsonb | issue id, number, actor name, org id |
| read_at | timestamptz nullable | |
| created_at | timestamptz | |

Denormalised payload on purpose: a notification should render without joining four tables, and it should still read correctly after the issue it refers to is renamed or deleted.

### AuditLog
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| organization_id | uuid FK → Organization | |
| actor_user_id | uuid FK → User | |
| action | text | e.g. `issue.status_changed` |
| resource_type | text | `issue`, `project`, `member`, … |
| resource_id | uuid | may point at a soft-deleted row |
| metadata | jsonb | e.g. `{ from: "TODO", to: "DONE" }` |
| created_at | timestamptz | |

**Immutable.** No `updated_at`, no update path, no delete path.

---

## 2. Relationships

```
User ─┬─< OrganizationMember >─── Organization ─┬─< Invitation
      │                                         │
      ├─< RefreshToken                          ├─< Project ─< Issue ─┬─< Comment
      │                                         │                     ├─< File
      ├─< Notification                          │                     └─> User (assignee)
      │                                         │
      └── (actor) ────────────────────────────< AuditLog
```

Every org-scoped row reaches `Organization` by a chain of foreign keys. That chain is what makes tenant isolation checkable: there is no org-scoped table you can reach without passing through `organization_id`.

---

## 3. Indexes — and the query each one exists for

| Index | Query it serves |
|---|---|
| `OrganizationMember (user_id, organization_id)` unique | The membership check on **every single org-scoped request**. The hottest query in the system |
| `OrganizationMember (organization_id)` partial, OWNER | Not a query — enforces the single-OWNER invariant |
| `OrganizationMember (organization_id)` | Member list for an org |
| `Invitation (organization_id, email)` partial, pending | Prevents duplicate pending invites |
| `Invitation (token_hash)` unique | Accept-invitation lookup by link |
| `Project (organization_id)` | Project list — every org dashboard load |
| `Project (organization_id, key)` unique | Key collision check on project creation |
| `Issue (project_id, status)` | **The board.** Issues in a project grouped by column — the single most frequent query in the app |
| `Issue (project_id, number)` unique | Direct lookup of `WEB-42`, plus the numbering safety net |
| `Issue (assignee_id)` | "My issues" across projects |
| `Issue` GIN on `search_vector` | Full-text search (Slice 9) |
| `Comment (issue_id, created_at)` | Comment thread, chronological |
| `File (issue_id)` | Attachments on an issue |
| `Notification (user_id, read_at)` | Unread count — runs on most page loads |
| `AuditLog (organization_id, created_at DESC)` | Activity timeline, newest first, paginated |

No index exists here without a query above it. Indexes get *verified* in Slice 9 with `EXPLAIN` against seeded data — an index nothing uses is a write-cost with no reader.

---

## 4. Concurrency

Two places where the naive implementation is wrong under concurrent use.

### 4.1 Per-project issue numbering

`SELECT max(number) + 1` then insert is a race: two simultaneous creations read the same max and collide.

Correct approach — a single atomic statement, inside the creation transaction:

```sql
UPDATE project
   SET last_issue_number = last_issue_number + 1
 WHERE id = $1
RETURNING last_issue_number;
```

One `UPDATE … RETURNING` takes a row lock and increments atomically; concurrent creators serialise on that row. The `unique (project_id, number)` constraint is the safety net that turns any remaining bug into an error instead of a duplicate.

### 4.2 Ownership transfer

The single-OWNER partial index is checked **per statement**, not deferred to commit. So the order inside the transaction matters:

```
BEGIN
  UPDATE member SET role = 'ADMIN'  WHERE …current owner…   -- frees the slot first
  UPDATE member SET role = 'OWNER'  WHERE …new owner…
COMMIT
```

Promote-then-demote fails: at the first statement there would momentarily be two OWNERs, and the index rejects it.

---

## 5. Transaction boundaries

| Operation | Must be atomic because |
|---|---|
| Accept invitation | Create membership + mark invitation accepted — a crash between them either loses the membership or leaves a reusable invitation |
| Create issue | Increment counter + insert issue — separately, a crash burns a number or duplicates one |
| Transfer ownership | Two role updates that transiently violate an invariant between them |
| Change password | Update hash + revoke all refresh tokens — otherwise old sessions survive a password change |
| Delete project | Soft-delete project + cascade soft-delete its issues |
| Delete organization | Soft-delete the org + soft-delete every project in it (which cascades to their issues) + hard-delete its `OrganizationMember` and pending `Invitation` rows — all in one transaction, so no half-deleted org is ever visible |

Everything else is a single statement and needs no explicit transaction.

---

## 6. Deletion strategy

| Entity | Strategy | Why |
|---|---|---|
| Organization, Project, Issue | Soft delete (`deleted_at`) | AuditLog references them by id after deletion; hard-deleting breaks the history that audit exists to provide |
| OrganizationMember, Invitation (on org deletion) | Hard delete, cascaded from the org | Meaningless without a live organization; unlike Project/Issue, nothing references a membership row by id afterward, so there's no audit reason to keep them |
| File | Soft delete metadata; object left in storage | Storage reclamation deferred — accepted cost, see ADR-0010 |
| Comment | Hard delete | Nothing references it afterward |
| Invitation (revoked) | Hard delete | Disposable |
| RefreshToken | Revoke (`revoked_at`), purge later | Revocation must be auditable in the moment |
| User | Not deletable in this product | Out of scope |

**Every query against a soft-deletable table must filter `deleted_at IS NULL`.** This is the standing cost of the strategy and the most likely place for a bug — it belongs in a Prisma middleware/extension applied centrally, not repeated by hand in every service.
