# ShipFlow — Documentation

A multi-tenant project and issue tracker for small software teams. These docs are written for the dev team: what we're building, why it's shaped this way, and what we deliberately said no to.

## Start here

| Doc | What's in it |
|---|---|
| [PRODUCT.md](./PRODUCT.md) | Who it's for, the core loop, every capability a user has, guardrails, non-goals |
| [UX.md](./UX.md) | User flows, wireframes, screen inventory, empty and failure states |
| [ROADMAP.md](./ROADMAP.md) | The eleven build slices, MVP line, definition of done |

## Design

| Doc | What's in it |
|---|---|
| [../CONTEXT.md](../CONTEXT.md) | The glossary. Domain vocabulary — use these words everywhere |
| [DATA-MODEL.md](./DATA-MODEL.md) | Entities, constraints, indexes and the query each serves, concurrency, deletion |
| [API.md](./API.md) | REST conventions, error format, pagination, endpoint map |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System shape, module boundaries, request lifecycle, async, caching, failure behaviour |
| [SECURITY.md](./SECURITY.md) | Threat model, controls, the cross-origin cookie problem |
| [TESTING.md](./TESTING.md) | Test pyramid, priority order, what we don't test |
| [CONVENTIONS.md](./CONVENTIONS.md) | Folder structure, naming, environments, git workflow |
| [adr/](./adr/) | Individual decisions with their rationale and rejected alternatives |

## The short version

**Product.** A tech lead on a 6-person team tracks work across projects, with real roles and real history, without Jira's setup tax. Organization is the tenant boundary; everything lives behind membership.

**Architecture.** Modular monolith plus a worker. Next.js → NestJS → Postgres, with Redis for caching and queues and R2 for files. Postgres is the only dependency allowed to take the product down; everything else degrades a feature.

**Approach.** Eleven vertical slices, each shipping schema through UI with its own tests. MVP at Slice 4, flagship at Slice 10, deployed from Slice 1.

## Decisions

Fourteen ADRs in [`adr/`](./adr/). The ones that shape the most code:

- [0001](./adr/0001-multi-org-membership.md) — users belong to many organizations
- [0003](./adr/0003-implicit-project-access.md) — org membership grants project access; no per-project membership
- [0004](./adr/0004-jwt-access-refresh-auth.md) — JWT + rotated refresh tokens over server sessions
- [0007](./adr/0007-org-id-in-url-path.md) — organization id in the URL, one guard for every route
- [0008](./adr/0008-soft-delete-convention.md) — soft delete where audit history references the row
- [0012](./adr/0012-redis-cache-fail-open.md) — Redis is cache-only and fails open
