# ShipFlow — Design Document

This is the synthesized output of the grilling session recorded in `../CONTEXT.md` (glossary) and `adr/` (individual decisions with rationale). This document organizes those decisions into a buildable plan. Where a decision needs its "why," this doc points at the ADR rather than re-arguing it.

## 1. Product definition

ShipFlow is a multi-tenant project and issue tracker for small software teams. A user signs up, creates or joins an Organization, invites teammates, creates Projects, and tracks Issues through a fixed workflow — collaborating via comments, file attachments, and notifications, with a full activity history.

**Non-goals** (explicitly not building): configurable per-project workflows, real-time multi-user collaboration, per-project membership restrictions, multiple issue assignees, email notifications beyond invitations, public API/webhooks, billing, SSO/OAuth. Each of these is a plausible, bounded future slice — not a gap in the MVP.

## 2. MVP scope

**Must-have**: auth (register/login/refresh), organizations, org membership + roles (OWNER/ADMIN/MEMBER), invitations, projects, issues (fixed status, single assignee, structured filters + full-text search), comments, file attachments, in-app notifications, audit log, rate limiting, caching (membership lookups), background jobs (invitation email), health checks, structured logging, error tracking, CI/CD, containerized deployment.

**Nice-to-have (build only if time remains after must-haves + their tests)**: password reset flow, "remove and re-invite" UX polish, audit log filtering by actor/date in the UI.

**Explicitly not built** (see Non-goals above and the corresponding ADRs 0001–0003): real-time board updates, configurable workflows, per-project membership, multi-assignee issues, OAuth login, email notifications beyond invites.

## 3. Domain model

Full glossary: `../CONTEXT.md`. Relationships:

```
User ──< OrganizationMember >── Organization ──< Invitation
                                     │
                                     ├──< Project ──< Issue ──< Comment
                                     │                  │
                                     │                  ├──< File
                                     │                  └──> User (assignee, nullable)
                                     │
                                     └──< AuditLog

User ──< Notification
```

- `OrganizationMember` is the join carrying `Role`; `User` has no direct link to `Organization`.
- `Project` and `Issue` are soft-deletable (ADR-0008); `Invitation` is hard-deleted on revoke/expiry.
- `Notification` belongs to a `User`, not an `Organization` — a notification about an org event is still fundamentally "for this person."

## 4. Architecture

```
Next.js (Vercel) ──HTTPS──> NestJS API (Render)
                                  │
                    ┌─────────────┼─────────────┐
                    v             v             v
              PostgreSQL      Redis        Cloudflare R2
               (Neon)       (Upstash)     (object storage)
                                  │
                                  v
                            BullMQ Queue
                                  │
                                  v
                         Worker (Render, separate service)
```

Modular monolith: one NestJS deployable for the API, one separate deployable for the worker, sharing the same codebase and Prisma client but scaled/restarted independently. No microservices, no message broker beyond BullMQ/Redis (ADR context: your original constraints ruled out Kafka/K8s/service meshes — nothing here needed them).

## 5. Backend module boundaries

| Module | Owns | Notes |
|---|---|---|
| `auth` | registration, login, JWT issuance/refresh, password hashing | No knowledge of orgs |
| `users` | user profile CRUD | Thin — most identity work happens via `organizations` |
| `organizations` | org CRUD, membership, role changes, ownership transfer | Owns the tenant-isolation guard |
| `invitations` | invite/accept/revoke/resend | Depends on `organizations` + `jobs` (email) |
| `projects` | project CRUD | Depends on `organizations` guard |
| `issues` | issue CRUD, status transitions, assignment, search/filter | Depends on `projects` |
| `comments` | comment CRUD on issues | Depends on `issues` |
| `files` | signed upload URLs, file metadata, deletion | Depends on `issues`, talks to R2 |
| `notifications` | in-app notification creation/read | Triggered by `issues` (assignment) |
| `audit-log` | write + query audit entries | Triggered by every module that performs a tracked action |
| `jobs` | BullMQ producer/consumer wiring | Used by `invitations` (worker process entry point) |
| `health` | liveness/readiness | Checks `organizations`' DB connection + Redis |
| `shared` | Prisma client, guards, decorators, exception filters | Cross-cutting, no business logic |

`audit-log` is intentionally not owned by any single feature module — every module that performs a tracked action writes to it directly (a shared service, not an event bus) since we don't have a message broker and don't need one at this scale.

## 6. Database design

| Table | Key columns | Constraints | Notes |
|---|---|---|---|
| `User` | id, email, passwordHash, name | `email` unique | |
| `Organization` | id, name, deletedAt | | |
| `OrganizationMember` | id, userId, organizationId, role | unique `(userId, organizationId)`; **partial unique `(organizationId)` WHERE `role = 'OWNER'`** | The partial index enforces "exactly one OWNER" at the DB level (ADR-0006), not just app logic |
| `Invitation` | id, organizationId, email, role, token, expiresAt, acceptedAt, revokedAt | unique `token`; **partial unique `(organizationId, email)` WHERE pending** | Prisma's schema DSL doesn't express partial unique indexes directly — this needs a raw-SQL migration addition after `prisma migrate dev` scaffolds the table |
| `Project` | id, organizationId, name, key, deletedAt | unique `(organizationId, key)` | `key` is the short prefix for issue numbers, e.g. `PROJ` |
| `Issue` | id, projectId, number, title, description, status, assigneeId, reporterId, deletedAt | unique `(projectId, number)` | `number` is a per-project sequential integer — see Concurrency below |
| `Comment` | id, issueId, authorId, body, editedAt, deletedAt | | Hard-deleted on removal — not in the audit-log event list, no downstream reference to preserve |
| `File` | id, issueId, uploaderId, fileName, mimeType, sizeBytes, storageKey, deletedAt | | |
| `Notification` | id, userId, type, payload (json), readAt | | |
| `AuditLog` | id, organizationId, actorUserId, action, resourceType, resourceId, metadata (json), createdAt | | Immutable — no `updatedAt`, no update path |

### Indexes and the query each one supports

- `OrganizationMember(userId, organizationId)` unique — the tenant-isolation guard's hot-path lookup, run on every org-scoped request (ADR-0007), and the Redis cache-miss fallback (ADR-0012).
- `OrganizationMember(organizationId)` partial WHERE `role='OWNER'` — invariant enforcement, not a query optimization.
- `Project(organizationId)` — "list projects in this org," the org dashboard's primary query.
- `Issue(projectId, status)` — the Kanban board's core query: "issues in project X, grouped by status."
- `Issue(assigneeId)` — "my issues" view, cross-project.
- `Issue` generated `tsvector` column + GIN index — full-text search (ADR-0009).
- `Comment(issueId, createdAt)` — ordered comment fetch per issue.
- `Notification(userId, readAt)` — unread-count query for the bell icon, run on most page loads.
- `AuditLog(organizationId, createdAt)` — reverse-chronological activity feed.

### Concurrency and transactions

Two places need explicit thought, not just "wrap it in a transaction":

1. **Per-project issue numbering.** Two simultaneous issue creations in the same project both computing `MAX(number) + 1` can race and collide. Fix: a `SELECT ... FOR UPDATE` on a per-project counter (or a Postgres sequence per project, created alongside the project) inside the same transaction as the insert — not a naive read-then-write.
2. **Ownership transfer** (ADR-0006). Because the partial unique index is checked per-statement, the transaction must demote the current OWNER to ADMIN *before* promoting the new member to OWNER, both within one transaction — reversing the order would transiently violate the single-owner constraint and fail.

## 7. Authentication and authorization

- **AuthN**: email/password, JWT access token (~15 min) + rotated refresh token (httpOnly cookie) (ADR-0004). No OAuth, no email verification gate for MVP.
- **AuthZ**: a NestJS guard resolves `:orgId` from the URL path (ADR-0007), looks up the caller's `OrganizationMember` row (cached per ADR-0012), and a role-checking decorator (`@RequireRole('ADMIN')`) gates specific routes. IDOR prevention is structural: every org-scoped query is additionally filtered by the `organizationId` the guard already verified — a resource ID alone is never trusted to imply access.

### Role permission matrix (from Q8)

| Action | OWNER | ADMIN | MEMBER |
|---|---|---|---|
| Invite/remove members, change roles | ✅ | ✅ | ❌ |
| Transfer ownership | ✅ (source) | ❌ | ❌ |
| Create/delete Project | ✅ | ✅ | Create only |
| Create/edit/assign Issue | ✅ | ✅ | ✅ |
| Delete Issue | ✅ | ✅ | own-reported only *(assumption — flag if wrong)* |
| Comment CRUD (own) | ✅ | ✅ | ✅ |
| Delete others' Comment | ✅ | ✅ | ❌ |

The "MEMBER delete Issue" row is my own call, not something we explicitly grilled — flagging it so you can override before Phase 5.

## 8. Multi-tenancy strategy

Organization is the sole isolation boundary (ADR-0001, ADR-0003). Enforcement is application-level (the guard described above), not Postgres row-level security — RLS would add real value at a much larger scale or with untrusted direct-DB access, neither of which applies here; a single, well-tested guard covers every route uniformly and is far easier to reason about and unit-test than RLS policies.

## 9. API design

REST, JSON, org ID in the URL path. Pagination: **cursor-based** (keyset, on `(createdAt, id)`), not offset — offset pagination drifts under concurrent writes and degrades on large tables; this wasn't explicitly grilled but is a low-controversy default worth naming rather than silently picking. Errors: a consistent envelope `{ statusCode, error, message, details? }`, `details` populated for 400s with field-level validation errors (class-validator).

### Representative endpoint map (not exhaustive — grown per phase, not built upfront)

```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout

POST   /organizations
GET    /organizations
GET    /organizations/:orgId
POST   /organizations/:orgId/transfer-ownership

GET    /organizations/:orgId/members
PATCH  /organizations/:orgId/members/:memberId
DELETE /organizations/:orgId/members/:memberId

POST   /organizations/:orgId/invitations
DELETE /organizations/:orgId/invitations/:id
POST   /invitations/:token/accept          ← not org-nested; acceptor has no access yet

POST   /organizations/:orgId/projects
GET    /organizations/:orgId/projects
PATCH  /organizations/:orgId/projects/:id
DELETE /organizations/:orgId/projects/:id

POST   /organizations/:orgId/projects/:projectId/issues
GET    /organizations/:orgId/projects/:projectId/issues?status=&assigneeId=&q=&cursor=
PATCH  /organizations/:orgId/projects/:projectId/issues/:id
DELETE /organizations/:orgId/projects/:projectId/issues/:id

POST   /.../issues/:id/comments
PATCH  /.../comments/:id
DELETE /.../comments/:id

POST   /.../issues/:id/files/upload-url    ← returns signed PUT URL + fileId
POST   /.../files/:fileId/complete         ← client confirms upload finished
DELETE /.../files/:fileId

GET    /notifications
POST   /notifications/:id/read

GET    /organizations/:orgId/audit-log?cursor=

GET    /health
GET    /health/ready                        ← checks Postgres + Redis
```

## 10. Async job architecture

BullMQ + Redis, one job type for MVP: `send-invitation-email`, keyed deterministically by Invitation ID for idempotency (ADR-0011). Flow: API creates `Invitation` row → enqueues job → responds → worker (separate process) sends email → marks done. Retries: BullMQ default exponential backoff, capped attempts, failed jobs visible via BullMQ's own state (inspectable in a small admin script or the Bull Board UI if we add it — optional, not committed to the roadmap).

## 11. File upload architecture

Client requests a signed upload URL from the API (which checks org/issue access first) → client `PUT`s directly to R2 → client confirms completion → API writes the `File` metadata row. Never proxy file bytes through the API. Limits: 10MB, MIME allowlist (ADR context: Q15). Soft-deleted Issues leave their File objects in storage (ADR-0010) — an accepted, explicitly deferred cost.

## 12. Testing strategy

Real Postgres for integration tests, scoped Playwright E2E (ADR-0014). Priority order: authorization/tenant-isolation tests first (these are the tests whose failure would be the worst kind of bug), then business-rule tests (role matrix, status transitions, per-project numbering under concurrency), then the golden-path E2E suite.

## 13. Security strategy

| Concern | Mitigation |
|---|---|
| IDOR | Every query scoped by the guard-verified `organizationId`, never trusts a bare resource ID (§8) |
| Password handling | argon2/bcrypt hash, never logged, never returned in any response |
| Token theft | Short-lived access token; refresh token httpOnly + rotated; ADR-0004 names the accepted revocation-latency tradeoff |
| Input validation | class-validator DTOs on every endpoint; reject unknown fields |
| Rate limiting | IP-keyed on pre-auth routes, user-keyed on authenticated routes (§ Q19), Redis-backed |
| CORS | Allowlist the deployed frontend origin only |
| File upload abuse | Size cap, MIME allowlist, signed URLs scoped to one object |
| Secrets | Environment variables only, never committed; documented in `.env.example` |
| Error leakage | Global exception filter strips stack traces / internal messages in production responses; full detail goes to Sentry, not the client |

## 14. Observability strategy

Structured JSON logs (pino), request/error logging via NestJS interceptor, `/health` (liveness) and `/health/ready` (Postgres + Redis connectivity), Sentry for production error tracking (ADR-0013). The test: "what would I need to diagnose this at 2am" — a failed readiness check plus a Sentry stack trace should answer "what broke and where" for the large majority of incidents at this scale.

## 15. Local development architecture

`docker-compose.yml`: Postgres, Redis, and (optionally) a local S3-compatible container for offline file-upload testing. API and web run natively (`pnpm dev`) against the composed infra for fast iteration; worker runs as a third `pnpm` process. `.env.example` documents every variable.

## 16. Deployment strategy

Vercel (web) + Render (api, worker) + Neon (Postgres) + Upstash (Redis) + Cloudflare R2 (storage) + Sentry (errors) — all free-tier (ADR-0013). GitHub Actions: lint → typecheck → unit+integration tests (Postgres service container) → build → deploy on merge to `main`.

## 17. Git workflow

Small, conventional commits (`feat(auth): ...`, `test(authz): ...`) per your original spec. One PR per vertical slice (roadmap phase below), not per file.

## 18. Repository structure

```
shipflow/
├── apps/
│   ├── web/           Next.js
│   └── api/           NestJS (includes the worker entrypoint)
├── packages/
│   └── shared/        Shared TS types (DTOs, enums) used by both apps
├── docs/
│   ├── DESIGN.md       (this file)
│   └── adr/
├── CONTEXT.md
├── docker-compose.yml
└── .github/workflows/
```

## 19. Major decisions and tradeoffs

All recorded as ADRs in `docs/adr/`: 0001 multi-org membership, 0002 fixed issue status, 0003 implicit project access, 0004 JWT auth, 0005 monorepo, 0006 single owner + transfer, 0007 org-id-in-path, 0008 soft delete, 0009 Postgres full-text search, 0010 orphaned file objects, 0011 BullMQ idempotency, 0012 Redis fail-open cache, 0013 deployment stack, 0014 testing strategy.

## 20. Phased roadmap (vertical slices)

| Phase | Slice | Key tests |
|---|---|---|
| 0 | Scaffolding: monorepo, Docker Compose, Prisma init, CI skeleton | CI green on an empty test |
| 1 | Auth end-to-end | register/login/refresh, invalid/expired token rejection |
| 2 | Organizations + membership + RBAC guard + Redis cache | **tenant isolation** (org A can't touch org B), role-gated actions |
| 3 | Invitations + BullMQ email job | idempotent job (no double-send), expired/duplicate invite rejected |
| 4 | Projects | authorization per role |
| 5 | Issues (status, assignee, per-project numbering) | concurrent creation doesn't collide numbers |
| 6 | Comments | author vs admin delete permissions |
| 7 | Files (signed URL flow) | cross-org access rejected, MIME/size validation |
| 8 | Full-text search + filters | correct results, index actually used (`EXPLAIN`) |
| 9 | Notifications | scoped to correct user only |
| 10 | Audit log | each tracked action produces exactly one correct entry |
| 11 | Rate limiting | 429 after threshold, resets after window |
| 12 | Observability (logs, health, Sentry) | readiness fails correctly when DB is stopped |
| 13 | CI/CD + Docker + deployment | push to `main` deploys and works live |
| 14 | Playwright E2E golden paths | the 5 core journeys pass |

Each phase ends in a working, deployed-if-relevant state before the next starts, per your original vertical-slice instruction.

## 21. Definition of Done (applies to every phase)

- Feature works end-to-end (API +, where relevant, UI) against real infra, not mocks
- Tests for the phase's "Key tests" column pass
- `lint` + `typecheck` clean
- Relevant module's authorization checked by a test, not just code review
- Commit(s) follow the conventional format
- Anything hard-to-reverse and non-obvious decided along the way gets an ADR

## 22. Risks / where we're likely to over-engineer

- **RBAC**: tempting to add a permissions framework (CASL etc.) before three roles actually strain a hand-rolled guard. Don't, until the guard genuinely gets unwieldy.
- **Notifications**: tempting to generalize into a pub/sub fanout system before there's more than one event type. Resist until Phase 9 proves the simple version is actually insufficient.
- **Observability**: tempting to reach for a full metrics/tracing stack (Prometheus/Grafana). Sentry + structured logs + health checks is the right size for this project; naming this now so it doesn't creep later.
- **Search**: tempting to reach for Elasticsearch "for real search." Explicitly ruled out (ADR-0009) — Postgres FTS is the right size.

## 23. Open questions requiring approval

None outstanding — all four grilling rounds are closed. Two spots worth a second look before their phase starts: the "MEMBER can only delete own-reported issues" call in §7, and whether Comment should be soft- rather than hard-deleted (currently hard-delete, §6) — both are my calls, not yours, flagged for override.
