---
status: accepted
---

# Issue search uses Postgres full-text search, not a separate search engine

Free-text search over Issue title/description is a generated `tsvector` column with a GIN index, queried alongside plain structured filters (status, assignee, project) — not Elasticsearch or another external search service. We considered plain `ILIKE` (simpler, no new column, but no ranking and slow past a small row count) and an external search engine (explicitly ruled out by project scope). Postgres full-text search is the deliberate middle ground: real indexing and ranking behavior worth demonstrating, without adding a service to operate.
