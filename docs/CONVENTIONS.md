# ShipFlow — Engineering Conventions

Folder structure, naming, environments, and git workflow. The point of a convention is that nobody has to think about it twice.

---

## 1. Repository layout

```
shipflow/
├── apps/
│   ├── api/                          NestJS — API + worker share this codebase
│   │   ├── src/
│   │   │   ├── modules/              one folder per domain concept
│   │   │   │   ├── auth/
│   │   │   │   ├── account/
│   │   │   │   ├── organizations/
│   │   │   │   ├── invitations/
│   │   │   │   ├── projects/
│   │   │   │   ├── issues/
│   │   │   │   ├── comments/
│   │   │   │   ├── files/
│   │   │   │   ├── notifications/
│   │   │   │   ├── audit/
│   │   │   │   └── health/
│   │   │   ├── common/               guards, decorators, filters, pipes, interceptors
│   │   │   ├── infra/                prisma, redis, storage, mailer, queue clients
│   │   │   ├── worker/               worker entrypoint + job processors
│   │   │   ├── app.module.ts
│   │   │   └── main.ts
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   └── test/                     integration tests (*.e2e-spec.ts)
│   └── web/                          Next.js
│       ├── src/
│       │   ├── app/                  App Router routes ONLY — no business logic
│       │   ├── features/             mirrors api/src/modules by name
│       │   │   └── issues/
│       │   │       ├── components/
│       │   │       ├── hooks/
│       │   │       └── api.ts
│       │   ├── components/ui/        shared primitives (Button, Dialog, …)
│       │   └── lib/                  api client, query client, utils
│       └── e2e/                      Playwright
├── packages/
│   └── shared/                       types, enums, API contracts used by both apps
├── docs/
├── docker-compose.yml
├── pnpm-workspace.yaml
└── .env.example
```

**The one rule worth remembering:** `web/src/features/*` mirrors `api/src/modules/*` by name. `features/issues` talks to `modules/issues`. Finding the other half of a feature is mechanical, never a search.

**A module folder looks like this** — flat until it hurts:

```
issues/
├── issues.module.ts
├── issues.controller.ts
├── issues.service.ts
├── issues.service.spec.ts
└── dto/
    ├── create-issue.dto.ts
    └── update-issue.dto.ts
```

No `repositories/` layer. Prisma *is* the repository; wrapping it in one adds indirection without adding a seam we need.

---

## 2. Naming

| Thing | Convention | Example |
|---|---|---|
| API files | kebab-case | `create-issue.dto.ts` |
| React components | PascalCase | `IssueCard.tsx` |
| Hooks | camelCase, `use` prefix | `useIssues.ts` |
| Classes | PascalCase + role suffix | `IssuesService`, `OrgMemberGuard`, `CreateIssueDto` |
| Prisma models | PascalCase singular | `Issue`, `OrganizationMember` |
| DB tables/columns | snake_case via `@@map`/`@map` | `organization_member`, `created_at` |
| API routes | plural nouns, kebab-case | `/organizations/:orgId/projects` |
| Env vars | SCREAMING_SNAKE | `DATABASE_URL`, `JWT_ACCESS_SECRET` |
| Job names | dot-namespaced | `invitation.send-email` |
| Audit actions | `resource.verb_past_tense` | `issue.status_changed` |
| Unit tests | colocated `*.spec.ts` | `issues.service.spec.ts` |
| Integration tests | `test/*.e2e-spec.ts` | `issues.e2e-spec.ts` |

**Language rules:** use the glossary in [`../CONTEXT.md`](../CONTEXT.md). It's an Issue, never a Task or Ticket. It's an Organization, never a Workspace, Team, or Tenant. This applies in code, tests, commits, UI copy, and conversation.

**Booleans** read as assertions: `isActive`, `hasExpired`, `canEdit` — never `active`, `expired`, `flag`.

**The snake_case DB mapping is deliberate.** We write raw SQL in two places (the search vector and the issue counter), and it should look like ordinary Postgres rather than a minefield of quoted `"createdAt"` identifiers.

---

## 3. Environments

Three. No more.

| Env | Infrastructure | Config |
|---|---|---|
| **local** | docker-compose Postgres + Redis; apps run natively (`pnpm dev`) | `.env`, gitignored |
| **test** | same compose infra, separate `shipflow_test` database, wiped per run | `.env.test`; CI uses service containers |
| **production** | Neon + Upstash + R2 + Render + Vercel | secrets in each platform's dashboard |

**Rules:**
- `.env.example` is committed and lists **every** variable with a comment explaining it. `.env` never is.
- Adding a variable means updating `.env.example` in the same commit. A teammate cloning the repo should never discover a missing variable at runtime.
- Local never points at the production database. Not once, not "just to check something."
- Migrations: `prisma migrate dev` locally, `prisma migrate deploy` in CI and production. Never `db push` outside a scratch branch.

**The setup contract** — clone to running in under five minutes, or the Foundation slice isn't finished:

```
git clone …
pnpm install
cp .env.example .env
docker compose up -d          # Postgres + Redis
pnpm --filter api prisma migrate dev
pnpm dev                      # web + api + worker
pnpm test                     # green
```

---

## 4. Git workflow

- **One branch per ticket**, named `slice-N/short-description` (e.g. `slice-2/org-rbac-guard`).
- **One PR per ticket**, merged before the next starts, so `main` is always in the working state the roadmap promises.
- **Conventional commits**, small and meaningful:
  ```
  feat(auth): add refresh token rotation
  fix(issues): prevent duplicate numbers under concurrent creation
  test(authz): add cross-tenant isolation cases
  chore(ci): run integration tests against postgres service
  ```
- **Never commit to `main` directly.** Never force-push a shared branch.
- A PR merges when CI is green. There is no "merge now, fix CI after."

---

## 5. Code style

- Prettier and ESLint run in CI; formatting is not a review topic.
- **TypeScript strict mode on.** No `any` without a comment explaining why — and "the types were annoying" isn't one.
- Comments explain **why**, never what. Code that needs a comment to explain what it does should be rewritten.
- Prefer flat over clever. This codebase gets read by someone (you, in six months, in an interview) explaining it out loud.
