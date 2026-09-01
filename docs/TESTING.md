# ShipFlow — Testing Strategy

Tests are written **with** each slice, never as a later phase. A test written after the code confirms what the code does; a test written alongside it confirms what the code *should* do.

---

## 1. The shape

```
      ╱╲        E2E (Playwright) — a handful of golden paths
     ╱  ╲       Slowest, most brittle. Only flows that must never break.
    ╱────╲
   ╱      ╲     Integration (Jest + Supertest + real Postgres)
  ╱        ╲    The bulk of the value. Real HTTP, real queries, real guards.
 ╱──────────╲
╱            ╲  Unit (Jest)
────────────── Pure logic: permission rules, validators, token handling.
```

Most projects invert this and mock the database. We don't, and the reason is specific: **the bugs that matter here are query bugs.** A missing `WHERE organization_id = …` is invisible to a mocked Prisma client and catastrophic in production. Integration tests run against real Postgres — Testcontainers locally, a service container in CI.

We do not chase a coverage number. Coverage measures lines executed, not behaviour verified.

---

## 2. Priority order

Written in this order, because this is the order of consequence:

1. **Tenant isolation.** For every org-scoped resource: a member of Org A gets 404 on Org B's resource. This is the test suite the whole product rests on.
2. **Authorisation.** Every row of the role matrix in [`PRODUCT.md`](./PRODUCT.md) §7, including the negative cases — MEMBER cannot invite, nobody can change the OWNER's role, OWNER cannot leave.
3. **Auth flows.** Register, login, refresh rotation, logout, expired token, reused refresh token, password change revoking sessions.
4. **Business invariants.** One OWNER per org, one pending invite per (org, email), invitation expiry, per-project issue numbering under concurrency.
5. **Critical API flows.** Create/read/update/delete for each resource, with validation failures.
6. **Async behaviour.** Job idempotency (double-enqueue sends one email), retry on failure.
7. **File authorisation.** No signed URL for another org's issue; size and MIME rejection.

---

## 3. What each layer covers

**Unit** — pure functions with no I/O: role comparison, DTO validators, token generation/verification, cursor encoding. Fast, colocated as `*.spec.ts`.

**Integration** — a real HTTP request through the real guard stack to a real database. Lives in `apps/api/test/*.e2e-spec.ts`. This is where tenant isolation, authorisation, and every endpoint's contract are actually verified. Each test seeds its own data and runs in a transaction rolled back afterwards, so tests don't leak into each other.

**E2E** — Playwright against a running stack, covering exactly the journeys whose breakage would make the product unusable:

1. Register → create org → create project → create issue → assign → move on the board
2. Invite → accept via link → second user sees the org
3. Comment on an issue and see it appear
4. Cross-tenant probe: a second org's board never shows the first org's issues

Four scenarios, not four hundred.

---

## 4. Concurrency and failure tests

Two tests worth writing carefully, because they're the ones that prove the design rather than the code:

- **Issue numbering under concurrency.** Fire N simultaneous issue creations at one project; assert N distinct sequential numbers and zero errors. This is the test that would catch a naive `MAX(number)+1`.
- **Redis unavailable.** Stop Redis; assert authorisation still works via the Postgres fallback and requests still succeed. This proves the fail-open behaviour is real rather than aspirational.

---

## 5. In CI

Every pull request runs: lint → typecheck → unit → integration (against a Postgres service container) → build. E2E runs on merge to `main`, not on every push — it's slow enough that gating every commit on it changes how often people push.

A failing test blocks the merge. There is no "fix it in the next PR."

---

## 6. What we deliberately don't test

- Framework behaviour (that NestJS routes, that Prisma queries)
- Third-party SDKs
- Trivial getters, or React components with no logic
- Exact copy in the UI — it changes constantly and testing it makes tests a chore rather than a safety net

Load testing arrives in Slice 9, once the system is complete enough for the numbers to mean anything.
