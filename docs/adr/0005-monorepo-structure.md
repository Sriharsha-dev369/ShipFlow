---
status: accepted
---

# Single monorepo for frontend, backend, and shared types

`apps/web` (Next.js), `apps/api` (NestJS), and a shared package for cross-cutting TypeScript types live in one repository (pnpm workspaces), rather than two independent repositories. We considered polyrepo — cleaner independent deploy cadence, closer to how separate teams would actually split ownership — but for a solo developer, sharing types between NestJS DTOs and the Next.js client, and running one CI pipeline that can reason about both sides of a change together, outweighs the deploy-independence polyrepo would buy. We're not pulling in Turborepo's build-caching machinery: at this scale it isn't earning its complexity yet.
