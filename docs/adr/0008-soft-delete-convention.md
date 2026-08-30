---
status: accepted
---

# Organization, Project, and Issue use soft delete; Invitation uses hard delete

Deleting an Organization, Project, or Issue sets `deletedAt` and excludes the row from normal queries, rather than removing it outright. We considered hard delete everywhere (simpler, relies on FK cascades) but rejected it for these three: AuditLog entries reference resources by ID after the action occurs (e.g. "Issue X was deleted"), and hard-deleting the very row an audit entry points to breaks that history. Invitation is the exception — a revoked or expired invitation is genuinely disposable and nothing else references it afterward, so it's hard-deleted. The accepted cost: every query against these three tables must remember to filter `deletedAt IS NULL`.
