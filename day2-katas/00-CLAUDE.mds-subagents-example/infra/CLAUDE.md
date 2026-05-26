# Infrastructure — CLAUDE.md

## Structure

```
infra/
  edge/
    worker.ts            # Cloudflare Worker entry point
    wrangler.toml        # Worker config, KV bindings, env vars
    routes.ts            # Edge routing logic
  ci/
    # Actual workflows live in .github/workflows/ — this dir has reusable scripts
    seed-test-db.sh      # Provision test DB for CI
    deploy.sh            # Deployment script called by workflows
  docker-compose.yml     # Local dev: Postgres + Redis
  docker-compose.test.yml # CI/test: isolated Postgres + Redis
  Dockerfile             # Multi-stage production build
```

## Edge (Cloudflare Workers)

- Worker handles: geo-routing, auth token verification, cache headers, static asset serving.
- Worker does NOT contain business logic — it proxies to the backend API.
- Shared types between edge and backend go in `src/shared/types/`. Do not import from `src/backend/`.
- KV is used for feature flags and edge config. Write via API, read at edge.
- Test with `wrangler dev --local` before deploying.
- Deploy: `wrangler deploy` (automated via CI on merge to main).

## Docker

- `Dockerfile` uses multi-stage: `deps` (install) -> `build` (compile) -> `runtime` (minimal image).
- Base image: `node:20-slim`. Final image should be under 200MB.
- Don't install dev dependencies in the runtime stage.
- `docker-compose.yml` for local dev mounts source code as volumes for hot reload.

## CI/CD (GitHub Actions)

Workflows in `.github/workflows/`:

| Workflow              | Trigger                  | Steps                                    |
|-----------------------|--------------------------|------------------------------------------|
| `ci.yml`              | PR to develop/main       | lint, typecheck, unit tests              |
| `deploy-staging.yml`  | merge to develop         | build, integration tests, deploy staging |
| `deploy-prod.yml`     | merge to main            | build, deploy prod (manual approval)     |
| `preview.yml`         | PR (frontend changes)    | deploy preview to Cloudflare Pages       |

- CI secrets are in GitHub repo settings. Never echo secrets in logs.
- Cache `node_modules` and Turborepo cache in CI for speed.
- Integration tests in CI use `docker-compose.test.yml` for DB + Redis.
- Deployment uses Docker image pushed to GitHub Container Registry.

## Making infra changes

- **New env var**: Add to `wrangler.toml` (edge) or Dockerfile (backend). Update `src/backend/config/env.ts` schema. Add to GitHub Actions secrets if sensitive.
- **New CI step**: Add to the appropriate workflow. Reuse scripts from `infra/ci/` where possible.
- **Edge routing change**: Edit `infra/edge/routes.ts`. Test locally with `wrangler dev --local`.
- **DB version bump**: Update `docker-compose*.yml` and `Dockerfile`. Test migration compat.
