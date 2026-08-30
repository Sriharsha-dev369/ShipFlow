---
status: accepted
---

# Authentication uses JWT access + rotated refresh tokens, not server-side sessions

Auth is a short-lived JWT access token plus a long-lived, rotated refresh token (stored as an httpOnly cookie), rather than an opaque server-side session cookie backed by a sessions table. We considered server-side sessions — simpler to revoke instantly, no rotation logic to get wrong — but chose JWT+refresh because it decouples the API from session storage (relevant once a separate worker process needs to validate the same auth context) and because rotation/revocation/"what if a refresh token leaks" is a deliberately instructive pattern for this project's stated learning goals. The accepted cost: revocation is not instant (an access token is valid until it expires, typically ~15 min) unless we add a token blocklist later.
