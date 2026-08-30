---
status: accepted
---

# Issue status is a fixed global enum, not a per-project configurable workflow

`Issue.status` is a fixed enum (e.g. `TODO`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`) shared by every Project, not a per-project `WorkflowStatus` table with custom ordering. We considered configurable workflows (the Linear/Trello "customize your columns" feature) but rejected it for MVP: it turns every status read into a join, adds a full CRUD surface for managing workflows, and doesn't serve this product's stated scope (a small team tool, not a workflow-configuration platform). Migrating from enum to a per-project table later is a real but bounded schema migration — worth deferring rather than paying for upfront.
