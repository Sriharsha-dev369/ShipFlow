# ShipFlow — Architecture

How the system is put together and why. Decisions with lasting consequences live in [`adr/`](./adr/); this document is the map.

---

## 1. System shape

```
                    Browser
                       │
                       │ HTTPS
                       ▼
              ┌─────────────────┐
              │  Next.js (web)  │   Vercel
              └────────┬────────┘
                       │  JSON over HTTPS
                       ▼
              ┌─────────────────┐
              │  NestJS (api)   │   Render
              └───┬────┬────┬───┘
                  │    │    │
      ┌───────────┘    │    └──────────────┐
      ▼                ▼                   ▼
┌───────────┐    ┌──────────┐       ┌─────────────┐
│ PostgreSQL│    │  Redis   │       │Cloudflare R2│
│  (Neon)   │    │(Upstash) │       │  (objects)  │
└───────────┘    └────┬─────┘       └─────────────┘
   source of          │ BullMQ queue
   truth              ▼
              ┌─────────────────┐
              │ Worker (same    │   Render
              │ codebase, own   │
              │ process)        │
              └─────────────────┘
```

**Modular monolith plus a worker.** One codebase, two deployables: the API process and the worker process. They share the Prisma client and the domain modules; they differ only in entry point. This is not microservices, and the reason is scale — at 3–15 users per organization, the operational cost of independent services buys nothing that module boundaries inside one process don't already give us.

---

## 2. Backend module boundaries

Each module owns its domain concept, its persistence, and its HTTP surface. Cross-module access goes through the other module's service, never its repository or tables.

| Module | Owns | Depends on |
|---|---|---|
| `auth` | Registration, login, tokens, password hashing | — |
| `account` | Profile and password changes | `auth` |
| `organizations` | Orgs, membership, roles, ownership transfer, **the tenant guard** | `auth` |
| `invitations` | Invite lifecycle, acceptance | `organizations`, `jobs` |
| `projects` | Project CRUD | `organizations` |
| `issues` | Issue CRUD, numbering, status, search | `projects` |
| `comments` | Comments on issues | `issues` |
| `files` | Signed URLs, file metadata | `issues`, `storage` |
| `notifications` | In-app notifications | — (written to by others) |
| `audit` | Audit entries and timeline | — (written to by others) |
| `jobs` | Queue producers, worker processors | `infra/redis` |
| `health` | Liveness, readiness | `infra` |
| `common` | Guards, decorators, filters, pipes | — |
| `infra` | Prisma, Redis, R2, mailer clients | — |

**`audit` and `notifications` are written to by many modules and depend on none.** They are leaf services called directly, not an event bus — we have no broker, and introducing one to decouple eleven modules that all run in the same process would be architecture for its own sake.

---

## 3. Request lifecycle

Every authenticated, org-scoped request runs the same pipeline. This uniformity is the tenant-isolation guarantee.

```
Request
  │
  ▼
CORS + rate limit            ← 429 if over budget
  │
  ▼
JwtAuthGuard                 ← validates access token → req.user     (401)
  │
  ▼
OrgMemberGuard               ← reads :orgId from the path
  │                            looks up membership (Redis, else Postgres)
  │                            → req.membership { role }              (404 if not a member)
  ▼
@RequireRole('ADMIN')        ← role check where the route needs one   (403)
  │
  ▼
ValidationPipe               ← DTO validation, unknown fields rejected (422)
  │
  ▼
Controller → Service
  │            └─ every query additionally scoped by the verified organizationId
  ▼
Response  ── structured log line with requestId, userId, orgId, duration
```

Two things make IDOR structurally hard rather than a matter of discipline:

1. **The org comes from the URL, so the guard applies to every org-scoped route by pattern.** You cannot add an endpoint under `/organizations/:orgId/...` that skips membership verification.
2. **Services scope by the *verified* org id from `req.membership`, never by an id from the request body.** A client-supplied `organizationId` is never trusted.

---

## 4. Caching

Redis is a cache and a queue backend. It is never a source of truth.

**What is cached:** exactly one thing — the `(userId, orgId) → role` membership lookup, keyed `member:{userId}:{orgId}`, short TTL, invalidated on role change, member removal, and org deletion.

**Why only that:** it's the one query that runs on *every single org-scoped request*. Caching issue lists or project lists would mean inventing a staleness tolerance for data users expect to be current, in exchange for saving a query that runs once per page rather than once per request.

**When Redis is down:** the guard falls back to Postgres and the request succeeds. **Fail open to the database, never fail closed.** Redis being unavailable degrades latency; it must never lock a team out of their own organization. (ADR-0012)

---

## 5. Asynchronous work

```
API: create Invitation ──► enqueue job (id = invitation id) ──► respond 201
                                       │
                                  Redis / BullMQ
                                       │
                            Worker: send invitation email
                                       │
                         success ──► done      failure ──► retry w/ backoff
                                                              │
                                                     exhausted ──► failed queue
```

**Idempotency is structural, not defensive.** A job's id is derived from the domain entity it acts on — the Invitation's id, not a random uuid. BullMQ then refuses to enqueue a duplicate job for the same invitation, so a retry after a crash, or an accidental double-enqueue from the API, cannot send the same email twice. (ADR-0011)

Retries use exponential backoff with a capped attempt count. Exhausted jobs land in the failed set rather than disappearing — a job that vanishes silently is worse than one that fails loudly.

**What runs async and what doesn't**: work goes on the queue only when it's slow, external, or allowed to fail independently of the request. Sending email qualifies. Writing an audit entry does not — it happens in the same transaction as the action it records, because an audit log that can silently lag or drop entries isn't an audit log.

---

## 6. File uploads

```
Client                        API                         R2
  │  "I want to upload X"      │                           │
  ├───────────────────────────►│                           │
  │                            │ authorise: member of org? │
  │                            │ validate: size, MIME      │
  │                            │ mint signed PUT URL       │
  │◄───────────────────────────┤ { fileId, uploadUrl }     │
  │                                                        │
  │  PUT bytes directly ──────────────────────────────────►│
  │                                                        │
  │  "upload complete"         │                           │
  ├───────────────────────────►│ mark File row complete    │
```

File bytes never transit the API process. Authorisation happens **before** the URL is minted, the storage key is server-generated (never derived from a user-supplied filename), and signed URLs are short-lived and scoped to a single object.

---

## 7. Frontend architecture

- **Next.js App Router**, TypeScript, Tailwind.
- **TanStack Query owns all server state.** No global store duplicating what the server already knows. Refetch on window focus is what makes the board feel live without a websocket.
- **`features/*` mirrors the API's `modules/*` by name** — `features/issues` talks to `modules/issues`. Navigating between the two halves of a feature is mechanical.
- **Optimistic updates only where rollback is obvious**: dragging a card between columns. Deletions and creations wait for the server.
- **The access token lives in memory, not `localStorage`** — an XSS that can read `localStorage` gets a long-lived credential; one that can only make requests is bounded by the token's lifetime. The refresh token is an httpOnly cookie the JS never sees.

---

## 8. Failure behaviour

What happens when each dependency is unavailable:

| Dependency down | Effect |
|---|---|
| **Postgres** | Hard outage. `/health/ready` reports 503, requests fail with 500. Nothing is served from cache as a substitute for truth |
| **Redis** | Degraded, not down. Membership checks hit Postgres; rate limiting fails open; new jobs can't be enqueued, so invitation *emails* stall while invitation *links* still work |
| **R2** | Uploads and downloads fail; everything else is unaffected |
| **Worker** | Emails queue up and send when it returns. Nothing is lost — the queue is durable |
| **Sentry / mailer** | Invisible to users. Never block a request on telemetry |

The pattern: **only Postgres is allowed to take the product down.** Every other dependency degrades a feature.
