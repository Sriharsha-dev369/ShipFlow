# ShipFlow — Build Roadmap

Eleven vertical slices. Each one leaves the application working, deployed, and demoable — never "finished except for the tests" or "done but not wired to the frontend."

**MVP = end of Slice 4. Flagship = end of Slice 10.**

---

## The principle

Every slice cuts through the whole stack:

```
schema ──► API ──► authorisation ──► frontend ──► tests ──► working feature
```

Not:

```
all schemas ──► all APIs ──► all frontend ──► tests ──► deploy      ✗
```

Three things are **properties of every slice**, never deferred to a later one: input validation, authorisation checks, and the tests that verify them. Only genuinely global work — an E2E suite, index tuning against real query patterns, a security review, load testing — waits for Slice 9.

---

## Slice 0 — Foundation

Reproducible engineering environment. No product features.

pnpm monorepo (`apps/api`, `apps/web`, `packages/shared`) · docker-compose Postgres + Redis · Prisma initialised · NestJS and Next.js booting · worker process entrypoint · structured logging and `/health` + `/health/ready` · GitHub Actions running lint, typecheck, and tests.

**Done when:** `git clone → docker compose up → migrate → pnpm dev → pnpm test` works in under five minutes on a clean machine, and CI is green.

---

## Slice 1 — Authentication → **first deploy**

Register, login, logout, refresh with rotation, protected API route, protected frontend route, account settings (name + password change, revoking other sessions).

Then **deploy the walking skeleton**: Vercel + Render + Neon, auto-deploy on merge to `main`.

Deploying here rather than at the end is the single most important sequencing decision in this roadmap. The cross-origin cookie problem ([`SECURITY.md`](./SECURITY.md) §3) surfaces the moment auth meets production — and here it surfaces when auth is the *only thing that exists*, instead of buried under ten slices.

**Done when:** a person can register and log in **on the deployed URL**, tests cover the auth flows, and every later slice deploys for free.

---

## Slice 2 — Organizations, membership, RBAC

Create an organization (creator becomes OWNER) · org switcher · the tenant-isolation guard on every org-scoped route · role decorator · member list · role changes · member removal · leave org · transfer ownership · org settings (rename, delete) · **invitation by link** — create an Invitation, copy the link manually. No email yet.

**Tenant isolation tests are written here, with the guard.** Not in Slice 9. This is the highest-consequence code in the project; everything above it assumes it works.

**Done when:** two real users exist in one organization with different roles, a user from another org gets 404 on everything, and the isolation suite passes.

---

## Slice 3 — Projects

Create, list, open, rename, soft-delete. Project settings page. Authorisation per the role matrix.

**Done when:** a user can log in, pick an organization, and manage its projects end to end.

---

## Slice 4 — Issues + Kanban → **MVP**

Issue CRUD · fixed status enum · priority · single assignee · per-project sequential numbering (concurrency-safe) · filtering, sorting, cursor pagination · Kanban board with drag-and-drop · issue detail page.

**Done when** the full core journey works on the deployed site:

```
sign up → create org → invite teammate → create project
       → create issue → assign it → move it across the board
```

**This is the MVP.** A real team could use it.

---

## Slice 5 — Comments

Add, edit own (with "edited"), delete own, admin delete of others'. Flat, chronological, paginated.

**Done when:** two users can hold a conversation on an issue and the permission rules hold.

---

## Slice 6 — Async jobs + notifications

BullMQ queue and worker process wired properly · **invitation emails** (closing the debt from Slice 2) · in-app notifications on assignment and on comments · bell with unread count · mark read.

Idempotency via deterministic job ids, retries with exponential backoff, failed-job visibility. Redis now has a reason to exist that isn't "SaaS apps use Redis."

**Done when:** an invitation arrives by email without the request waiting for it, a double-enqueue provably sends one email, and a killed worker resumes without losing jobs.

---

## Slice 7 — Files

Signed upload URLs against R2 · size and MIME validation · metadata in Postgres · signed download · delete.

**Done when:** a file attaches to an issue and downloads back, and a user from another org can't obtain a signed URL for it.

---

## Slice 8 — Audit & activity

Audit entries for every tracked action (member invited/removed, role changed, project created, issue created/assigned/status-changed/deleted) · organization activity timeline.

**Done when:** every tracked action produces exactly one correct entry — no more, no fewer — and the timeline reads as a truthful history.

---

## Slice 9 — Production hardening

The genuinely global work that couldn't be done per-slice:

Rate limiting (IP-keyed pre-auth, user-keyed authenticated) · Redis membership cache with fail-open · full-text search with `tsvector` + GIN · `EXPLAIN` on the hot queries · N+1 hunt · consistent error envelope audit · Playwright E2E golden paths · dependency audit · full security review against [`SECURITY.md`](./SECURITY.md).

**Done when:** the question shifts from "does it work?" to "what happens when it doesn't?" — and there's an answer for each dependency.

---

## Slice 10 — Observability & deployment hardening

Sentry · request/error logging with `requestId` correlation · uptime monitoring · rollback procedure · deployment and troubleshooting docs · production runbook.

**Done when:** you can answer "how would I diagnose this at 2am?" by pointing at something real, and a bad deploy can be rolled back without improvisation.

---

## Definition of Done — every slice

- [ ] Works end to end against real infrastructure, not mocks
- [ ] Deployed and working on the live URL (from Slice 1 onward)
- [ ] Validation and authorisation implemented **and tested**
- [ ] `lint` + `typecheck` + tests green in CI
- [ ] Conventional commits, one PR, merged to `main`
- [ ] Any hard-to-reverse, non-obvious decision recorded as an ADR
- [ ] You can explain the whole slice out loud: what problem, what alternatives, what tradeoff, what could go wrong

That last one is the real bar. If you can't explain it, the slice isn't done — it's just merged.
