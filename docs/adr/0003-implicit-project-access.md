---
status: accepted
---

# Project access is implicit from Organization membership

Any OrganizationMember can see and act on any Project within their Organization — there is no separate `ProjectMember` entity gating per-project access in MVP. We considered explicit per-project membership (matching how larger orgs restrict project visibility) but rejected it: it adds a second authorization layer (org role AND project membership) to every Issue operation, for a product explicitly scoped to small teams where the Organization itself is already a small, trusted unit. Adding `ProjectMember` later is an additive change (a new table plus a narrowing of an existing check), not a rework of what's already built.
