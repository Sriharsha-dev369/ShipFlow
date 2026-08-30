---
status: accepted
---

# Soft-deleting an Issue does not reclaim its File objects from storage

When an Issue is soft-deleted, its File metadata rows are soft-deleted alongside it, but the underlying objects in object storage are left in place rather than deleted. We considered deleting the storage objects immediately (cleaner, no orphaned storage cost) but rejected it for MVP: reclaiming storage on cascading soft-delete needs its own sweep (what if the Issue is never really gone, what if restore is added later) and isn't worth solving before it's a real cost problem. The accepted tradeoff: storage usage only ever grows; a reaper job to purge objects belonging to old soft-deleted Issues is explicitly deferred, not solved here.
