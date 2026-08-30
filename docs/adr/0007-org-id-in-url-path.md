---
status: accepted
---

# Organization ID is resolved from the URL path, not a header or token claim

Every org-scoped route is shaped `/organizations/:orgId/...`, and a guard checks the authenticated user's membership in `:orgId` on every request. We considered a header (`X-Org-Id`) or an "active organization" claim baked into the JWT, but rejected both: putting the org in the path keeps the tenant-isolation check at one uniform layer for every route (impossible to add a new org-scoped endpoint that accidentally skips it), and keeps the org visible in logs, caches, and URLs instead of hidden in a header. Frontend convenience (remembering the last-viewed org) is a client-side concern, not a server-side one.
