---
status: accepted
---

# Users belong to many Organizations via OrganizationMember

A User's relationship to an Organization is a separate `OrganizationMember` entity (many-to-many), not a single `organizationId` on `User`. We considered the simpler one-user-one-org model, but rejected it: it matches how real multi-tenant SaaS products work (org switching, being a MEMBER of one org and OWNER of another), and retrofitting multi-org membership after the schema, auth tokens, and every "current org" assumption in the API are built around single-org would be far more expensive than building it in from the start.
