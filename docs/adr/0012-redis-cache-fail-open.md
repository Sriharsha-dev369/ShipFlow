---
status: accepted
---

# Redis is cache-only and fails open

The only data cached in Redis is the per-request `(userId, orgId) → role` membership lookup used by the tenant-isolation guard, keyed `member:{userId}:{orgId}` with a short TTL, invalidated on role change/removal/org deletion. Postgres remains the source of truth. If Redis is unavailable or a key is missing, the guard falls back to querying Postgres directly rather than denying the request — we explicitly chose fail-open-to-the-database over fail-closed, since Redis being down should degrade latency, not take the product offline or lock everyone out of their own organization.
