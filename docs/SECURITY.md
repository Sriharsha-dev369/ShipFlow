# ShipFlow — Security

Each item names the **threat** and the **mitigation**. "We use HTTPS" is not a security posture; knowing what an attacker would try is.

---

## 1. Threat model in one line

The asset worth protecting is **one organization's data from every other organization**. The most damaging realistic attack is not a database breach — it's an authenticated user of Org A reading Org B's issues through a missing `WHERE` clause.

---

## 2. Controls

| Threat | Mitigation |
|---|---|
| **Cross-tenant read/write (IDOR)** | Org id comes from the URL and is verified by one guard on every org-scoped route; services scope queries by the *verified* org id, never a client-supplied one. Cross-tenant access returns **404, not 403**, so responses don't confirm a resource exists |
| **Credential stuffing / brute force** | 5 login attempts per 15 min per IP, Redis-backed. Identical error for wrong-email and wrong-password |
| **Password disclosure** | argon2id hashing. Passwords never logged, never returned, never in an error message |
| **Stolen access token** | Short lifetime (~15 min). Held in memory, not `localStorage` — an XSS gets a bounded credential rather than a durable one |
| **Stolen refresh token** | httpOnly, Secure, path-scoped cookie; rotated on every use. Reuse of a rotated token is detectable and revokes the family |
| **Session persistence after password change** | Changing a password revokes every refresh token for that user in the same transaction |
| **CSRF** | The refresh cookie is the only cookie-based credential; the access token travels in an `Authorization` header. See §3 — this needs care in production |
| **XSS** | React escapes by default; no `dangerouslySetInnerHTML`. Comment and description bodies are stored and rendered as plain text |
| **SQL injection** | Prisma parameterises everything. The two raw SQL sites (search vector, issue counter) use bound parameters, never string interpolation |
| **Mass assignment** | DTOs with `forbidNonWhitelisted` — unknown fields are rejected, not ignored. `role`, `organizationId`, `reporterId` are never client-settable on update |
| **Malicious file upload** | MIME allowlist (not a blocklist), 10 MB cap, server-generated storage keys, short-lived scoped signed URLs. Files are served from R2's origin, never the app's, so a stored HTML file can't execute in the app's origin |
| **Privilege escalation** | Role changes are ADMIN+ only; the OWNER's role can't be changed by anyone; single-OWNER enforced by a database constraint, not just service code |
| **Invitation abuse** | Tokens are random and stored hashed; 7-day expiry; one pending invite per (org, email); accepting always requires an explicit authenticated action |
| **Enumeration via invitation preview** | `GET /invitations/:token` reveals only the org name and role, and only to someone holding a valid unexpired token |
| **Error leakage** | One global exception filter. Stack traces and internal messages go to Sentry and the logs, never to the client |
| **Secret exposure** | Secrets only in environment variables, `.env` gitignored, `.env.example` documents names not values. No secrets in logs |
| **Dependency vulnerabilities** | `pnpm audit` in CI; Dependabot on the repo |
| **Runaway clients** | 100 req/min per authenticated user as a backstop against a buggy client, not as a business control |

---

## 3. The cross-origin cookie problem

**This is the security detail most likely to bite, and it's why deployment happens at Slice 1 rather than Slice 10.**

In local development the web app and API are both on `localhost`, so the refresh cookie is same-site and `SameSite=Lax` works. In production the web app is on Vercel and the API is on Render — **different registrable domains**. A `SameSite=Lax` cookie will simply not be sent, and auth breaks in a way that looks like a backend bug.

The options, and what each costs:

| Option | Consequence |
|---|---|
| `SameSite=None; Secure` cookie | Works cross-site — **but re-enables CSRF**, so it needs a paired mitigation (origin checking on the refresh endpoint, or a double-submit token) |
| Put the API behind the same apex domain (`app.shipflow.dev` + `api.shipflow.dev`) | Cookie can be `SameSite=Lax` on the shared parent domain; requires owning a domain and configuring both platforms |
| Proxy API calls through Next.js route handlers | Everything is same-origin from the browser's view; adds a hop and puts the web tier on the request path |

Decide this **when Slice 1 deploys**, with auth as the only thing that exists. Deferring it means discovering it with ten slices of code on top.

---

## 4. Practices, not features

- Authorisation is verified by **tests**, not by reading the code. Every org-scoped resource gets a "user from another org gets 404" test.
- New endpoints are reviewed against one question first: *what stops a member of another organization from calling this?*
- The `/invitations/:token` routes sit outside the uniform org guard by necessity. They get extra scrutiny in review, and they are the first place to look when something feels wrong.
- Security work that genuinely can't be done per-slice — a full review pass, dependency audit, `EXPLAIN` on the hot queries — happens in Slice 9. Per-endpoint validation and authorisation cannot be deferred there and never are.
