---
status: accepted
---

# Integration tests run against a real Postgres; E2E is scoped to golden paths only

Integration tests (authorization, tenant isolation, anything touching Prisma) run against a real Postgres instance (Testcontainers locally, a Postgres service container in CI), never a mocked ORM or SQLite. We considered mocking the database layer — faster, no infra needed — but rejected it specifically for authorization and tenant-isolation tests: those tests exist to catch real query bugs (a missing `WHERE organizationId = ...`), and a mock can't fail the way a real query can. Separately, Playwright E2E coverage is deliberately limited to a handful of golden-path journeys (signup → org → project → issue → invite), not broad UI coverage — E2E is the slowest, most brittle test layer, so it's reserved for the flows that must never break, with unit and integration tests carrying the bulk of coverage.
