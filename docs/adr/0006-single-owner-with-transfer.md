---
status: accepted
---

# Exactly one OWNER per Organization, moved via explicit transfer

An Organization has exactly one OWNER at any time; there is a dedicated "transfer ownership" action rather than allowing multiple simultaneous OWNERs. We considered allowing several OWNERs (simpler permission check: "is this role >= OWNER") but rejected it: "who is *the* owner" should be an unambiguous, single answer for actions like billing or org deletion, and a transfer action makes changing that answer a deliberate, auditable event rather than an implicit side effect of a role edit.
