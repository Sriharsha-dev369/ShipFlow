# ShipFlow

A multi-tenant project and issue tracker for small software teams. Organizations, projects, issues on a board, roles, and a real audit history — without the configuration tax of a large tool.

> **Status: in development.** Design is complete and documented; implementation starts at Slice 0. The setup instructions below describe the target contract, not a working install yet. Progress is tracked in [Issues](https://github.com/Sriharsha-dev369/ShipFlow/issues), sequenced by [`docs/ROADMAP.md`](./docs/ROADMAP.md).

---

## What it does

A person signs up, creates an organization, invites teammates with roles, creates projects, and tracks issues across a Kanban board — commenting, attaching files, getting notified, and seeing a history of what the team did.

The full capability list is in [`docs/PRODUCT.md`](./docs/PRODUCT.md). What it deliberately **doesn't** do — real-time collaboration, configurable workflows, per-project permissions, public sharing — is listed there too, each with the reason.

## Architecture

```
                    Browser
                       │ HTTPS
                       ▼
              ┌─────────────────┐
              │  Next.js (web)  │   Vercel
              └────────┬────────┘
                       │ JSON
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
  source of           │ BullMQ
   truth              ▼
              ┌─────────────────┐
              │     Worker      │   Render
              └─────────────────┘
```

A **modular monolith plus a worker**: one codebase, two deployables sharing domain modules and differing only in entry point. Not microservices — at this scale, module boundaries inside one process give the same separation without the operational cost.

## Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | Next.js, TypeScript, Tailwind, TanStack Query | TanStack Query owns server state; refetch-on-focus is what makes the board feel live without a websocket |
| Backend | NestJS, TypeScript, REST | Module system maps cleanly onto domain boundaries; decorators make guards composable |
| Database | PostgreSQL + Prisma | Relational data with real constraints; Prisma for migrations and type-safe access |
| Cache & queue | Redis + BullMQ | One cached lookup, one job queue — both with a stated reason to exist |
| Storage | Cloudflare R2 | S3-compatible, permanently free tier, no egress fees |
| Testing | Jest, Supertest, Playwright | Integration tests run against a **real** Postgres, never a mock |
| CI/CD | GitHub Actions | Lint, typecheck, test on every PR; deploy on merge |

## Engineering decisions worth reading

The parts that are actually interesting, each with its rationale and rejected alternatives in [`docs/adr/`](./docs/adr/):

- **Tenant isolation is structural, not disciplined.** The organization id lives in the URL path, one guard verifies membership on every org-scoped route, and services scope queries by the *verified* id — never one from the request body. Cross-tenant access returns **404, not 403**, so responses can't be used as a membership oracle. ([ADR-0007](./docs/adr/0007-org-id-in-url-path.md))
- **Redis is cache-only and fails open.** If Redis is down, authorisation falls back to Postgres and the product keeps working — slower, not broken. Only Postgres is allowed to cause an outage. ([ADR-0012](./docs/adr/0012-redis-cache-fail-open.md))
- **Background jobs are idempotent by construction.** Job ids derive from the domain entity, so the queue itself refuses duplicates — a retry after a crash cannot send the same email twice. ([ADR-0011](./docs/adr/0011-bullmq-idempotent-jobs.md))
- **Issue numbering is race-safe.** Per-project sequential numbers come from a single atomic `UPDATE … RETURNING` inside the creation transaction, with a unique constraint as the safety net — not `MAX(number) + 1`. ([`docs/DATA-MODEL.md`](./docs/DATA-MODEL.md) §4)
- **Exactly one OWNER, enforced by the database.** A partial unique index makes the invariant unbreakable, and ownership transfer must demote before promoting or the index rejects it. ([ADR-0006](./docs/adr/0006-single-owner-with-transfer.md))
- **Soft delete only where audit history needs it.** Organizations, projects, and issues stay referenceable after deletion because audit entries point at them; comments and revoked invitations are hard-deleted. ([ADR-0008](./docs/adr/0008-soft-delete-convention.md))

## Local development

```bash
git clone https://github.com/Sriharsha-dev369/ShipFlow.git
cd ShipFlow
pnpm install
cp .env.example .env

docker compose up -d                        # Postgres + Redis
pnpm --filter api prisma migrate dev        # schema
pnpm dev                                    # web + api + worker
pnpm test                                   # should be green
```

Clone to running in under five minutes is a hard requirement, not an aspiration — see [`docs/CONVENTIONS.md`](./docs/CONVENTIONS.md) §3.

## Documentation

| Doc | Contents |
|---|---|
| [docs/PRODUCT.md](./docs/PRODUCT.md) | Who it's for, every user capability, guardrails, non-goals |
| [docs/UX.md](./docs/UX.md) | Flows, wireframes, empty and failure states |
| [docs/DATA-MODEL.md](./docs/DATA-MODEL.md) | Entities, constraints, indexes and the query each serves, concurrency |
| [docs/API.md](./docs/API.md) | REST conventions, error format, pagination, endpoint map |
| [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) | Module boundaries, request lifecycle, caching, async, failure behaviour |
| [docs/SECURITY.md](./docs/SECURITY.md) | Threat model and controls |
| [docs/TESTING.md](./docs/TESTING.md) | Test strategy and priority order |
| [docs/CONVENTIONS.md](./docs/CONVENTIONS.md) | Folder structure, naming, environments, git workflow |
| [docs/ROADMAP.md](./docs/ROADMAP.md) | The eleven build slices |
| [CONTEXT.md](./CONTEXT.md) | Domain glossary |

## Roadmap

Eleven vertical slices, each leaving the app working and deployed. **MVP at Slice 4; flagship at Slice 10.**

| | Slice | Delivers |
|---|---|---|
| ☐ | 0 | Foundation — monorepo, local infra, CI, logging, health |
| ☐ | 1 | Authentication — **and the first production deploy** |
| ☐ | 2 | Organizations, membership, RBAC, invitations |
| ☐ | 3 | Projects |
| ☐ | 4 | Issues + Kanban board — **MVP** |
| ☐ | 5 | Comments |
| ☐ | 6 | Background jobs, invitation email, notifications |
| ☐ | 7 | File attachments |
| ☐ | 8 | Audit log and activity timeline |
| ☐ | 9 | Hardening — rate limiting, caching, search, performance, security, E2E |
| ☐ | 10 | Observability and deployment hardening |

Deployment lands at Slice 1 rather than the end on purpose: the app's auth cookie is cross-site in production (Vercel and Render are different domains) but same-site locally, so it works in development and silently fails in production. That's a problem worth meeting when auth is the only thing that exists.
