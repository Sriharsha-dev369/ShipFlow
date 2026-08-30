---
status: accepted
---

# Background jobs run on BullMQ, keyed for idempotency by domain ID

Async work (starting with `send-invitation-email`) runs on BullMQ against Redis, with each job's ID derived deterministically from the domain entity it acts on (e.g. the Invitation ID), not a random UUID. We considered a random job ID (simpler) but rejected it: a deterministic ID means BullMQ itself refuses to enqueue a duplicate job for the same Invitation, so a retry after a crash — or an accidental double-enqueue from the API layer — can't send the same email twice. This is the idempotency mechanism, not an afterthought bolted onto job processing.
