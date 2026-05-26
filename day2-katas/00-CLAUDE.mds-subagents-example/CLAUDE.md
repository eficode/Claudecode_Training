# CLAUDE.md — Acme Platform

## Project Overview

Full-stack SaaS platform: Next.js frontend, Node/Express API, PostgreSQL, Redis cache,
deployed to Cloudflare Workers (edge) with Docker-based CI/CD.

Monorepo managed with Turborepo. Node 20, pnpm 9.

## Quick Reference

| Layer       | Path              | Runtime         | Entry Point               |
|-------------|-------------------|-----------------|---------------------------|
| Frontend    | `src/frontend/`   | Next.js 14      | `src/frontend/app/`       |
| Backend API | `src/backend/`    | Express 5       | `src/backend/server.ts`   |
| DB          | `src/backend/db/` | Drizzle ORM     | `src/backend/db/schema.ts`|
| Cache       | `src/backend/cache/` | Redis 7 / ioredis | `src/backend/cache/client.ts` |
| Edge        | `infra/edge/`     | Cloudflare Workers | `infra/edge/worker.ts` |
| CI/CD       | `infra/ci/`       | GitHub Actions  | `.github/workflows/`      |
| Tests       | `tests/`          | Vitest + Playwright | `vitest.config.ts`    |

## Conventions

- **TypeScript strict mode** everywhere. No `any` unless interfacing with untyped externals.
- **Imports**: Use `@/` path aliases (`@/backend/...`, `@/frontend/...`). Never relative imports crossing layer boundaries.
- **Env vars**: All env access goes through `src/backend/config/env.ts` (validated with zod). Never read `process.env` directly.
- **Error handling**: Backend uses `AppError` class from `src/backend/errors.ts`. Frontend uses error boundaries per route segment.
- **Naming**: files `kebab-case.ts`, types `PascalCase`, functions `camelCase`, constants `UPPER_SNAKE`, DB tables `snake_case`.
- **No barrel files** (`index.ts` re-exports). Import from the actual module.

## Database

- **ORM**: Drizzle. Schema defined in `src/backend/db/schema.ts`.
- **Migrations**: `pnpm db:generate` creates migration, `pnpm db:migrate` applies. Never edit generated SQL by hand.
- **Queries**: Use Drizzle query builder. Raw SQL only in `src/backend/db/raw/` with a comment explaining why.
- **Transactions**: Use `db.transaction()` — never manually manage BEGIN/COMMIT.
- **Connection**: Pool managed in `src/backend/db/pool.ts`. Max 20 connections. Tests use a separate `_test` database.

## Cache (Redis)

- Client singleton in `src/backend/cache/client.ts`. All cache access goes through `src/backend/cache/store.ts`.
- Key format: `{service}:{entity}:{id}` e.g. `auth:session:abc123`.
- TTLs are constants in `src/backend/cache/ttl.ts`. Never hardcode TTL values.
- Cache invalidation happens in the same service that writes — no cross-service invalidation.

## Edge / Cloudflare Workers

- Worker code in `infra/edge/worker.ts`. Keep it minimal — routing, auth verification, cache headers.
- Edge cannot import from `src/backend/` directly. Shared types go in `src/shared/types/`.
- KV bindings defined in `infra/edge/wrangler.toml`.
- Test edge functions with `wrangler dev --local`.

## External Drivers & Integrations

- All third-party service clients live in `src/backend/drivers/`. Each driver is a thin wrapper exposing only what we use.
- Drivers: `stripe.ts` (payments), `resend.ts` (email), `s3.ts` (file storage), `openai.ts` (AI features).
- Every driver must have a mock in `tests/mocks/drivers/` for unit testing.

## Testing

- **Unit tests**: Colocated as `*.test.ts` next to source. Run with `pnpm test:unit`.
- **Integration tests**: In `tests/integration/`. Hit real DB (test instance) and real Redis. Run with `pnpm test:integration`.
- **E2E tests**: In `tests/e2e/`. Playwright against local dev server. Run with `pnpm test:e2e`.
- **Test DB**: Spun up via Docker Compose (`infra/docker-compose.test.yml`). `pnpm test:setup` provisions it.
- Always run `pnpm test:unit` after changes. Run `pnpm test:integration` when touching DB/cache code.

## CI/CD

- GitHub Actions workflows in `.github/workflows/`.
- `ci.yml`: lint + typecheck + unit tests on every PR.
- `deploy-staging.yml`: deploy to staging on merge to `develop`.
- `deploy-prod.yml`: deploy to prod on merge to `main`, requires approval.
- Docker images built with `infra/Dockerfile`. Multi-stage build: deps -> build -> runtime.
- Preview deployments on Cloudflare Pages for frontend PRs.

## Commands

```
pnpm dev              # Start all services locally (turbo)
pnpm build            # Production build
pnpm test:unit        # Unit tests
pnpm test:integration # Integration tests (needs Docker)
pnpm test:e2e         # E2E tests (needs dev server running)
pnpm test:setup       # Provision test DB + Redis
pnpm db:generate      # Generate migration from schema changes
pnpm db:migrate       # Apply pending migrations
pnpm db:studio        # Open Drizzle Studio
pnpm lint             # ESLint + Prettier check
pnpm typecheck        # tsc --noEmit across all packages
```

## Git Workflow

- Branch from `develop`. Name: `{type}/{ticket}-{short-desc}` e.g. `feat/AC-123-user-avatars`.
- Commit messages: conventional commits (`feat:`, `fix:`, `chore:`, `refactor:`, `test:`, `docs:`).
- PR must pass CI, have 1 approval, and squash-merge into `develop`.
- Release: `develop` -> `main` via release PR.

---

## Sub-Agent Workflow

When tackling complex tasks, split work across focused sub-agents using the Task tool.
This keeps context tight and lets independent work run in parallel.

### When to use sub-agents

- Task touches 2+ layers (e.g. frontend + backend + DB migration)
- Task involves both implementation and testing
- Research is needed before implementation (explore first, then code)
- CI/CD or infra changes that are independent of app code

### Agent dispatch patterns

**Feature spanning multiple layers:**
1. Explore agent — understand current code, find integration points
2. Backend agent — DB schema, API endpoints, cache logic (in parallel with frontend if API contract is agreed)
3. Frontend agent — UI components, data fetching
4. Test agent — unit + integration tests for the new code

**Bug fix:**
1. Explore agent — reproduce, find root cause
2. Fix agent — implement the fix
3. Test agent — add regression test, run existing suite

**DB migration + cache invalidation:**
1. Single backend agent — schema change, migration, update queries, update cache logic
2. Test agent — integration tests against test DB

### Agent instructions template

When spawning a sub-agent with the Task tool, include:
- Which layer it owns (backend / frontend / infra / tests)
- Specific files or directories to focus on
- The conventions from this CLAUDE.md that apply
- What output or artifacts to produce
- Reference the subfolder CLAUDE.md: "Read src/backend/CLAUDE.md for backend conventions"

### Parallel vs Sequential

Run in **parallel** when agents don't share files:
- Frontend + Backend (when API contract is pre-agreed)
- Unit tests + Lint/typecheck
- Infra changes + App code changes

Run **sequentially** when there are dependencies:
- DB migration THEN backend code THEN tests
- Backend API THEN frontend integration
- Implementation THEN integration tests
