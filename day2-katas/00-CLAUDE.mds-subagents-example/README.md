# Kata 00: CLAUDE.md Files & Sub-Agent Architecture (Reference)

A complete example of layered `CLAUDE.md` files for a team-sized full-stack project, with reusable sub-agent prompt templates and **per-agent model pinning** (2026 pattern).

The fictional project: **Acme Platform** — Next.js frontend, Express API, PostgreSQL, Redis, Cloudflare Workers edge layer, GitHub Actions CI/CD.

---

## Directory Structure

```
00-CLAUDE.mds-subagents-example/
├── CLAUDE.md                          # Root — project overview, conventions, commands,
│                                      #   git workflow, sub-agent dispatch patterns
├── .claude/
│   └── agents/                        # YAML-front-mattered, model-pinned specialists
│       ├── backend.md                 #   pinned: claude-sonnet-4-6
│       ├── frontend.md                #   pinned: claude-sonnet-4-6
│       ├── test.md                    #   pinned: claude-haiku-4-5  (cheap)
│       └── infra.md                   #   pinned: claude-sonnet-4-6
│
├── src/
│   ├── backend/
│   │   └── CLAUDE.md                  # Backend layer: route→service→DB/cache,
│   │                                  #   Drizzle ORM, Redis cache, driver wrappers
│   └── frontend/
│       └── CLAUDE.md                  # Frontend layer: Next.js App Router, shadcn/ui,
│                                      #   server vs client components, asset rules
├── tests/
│   └── CLAUDE.md                      # Testing: Vitest + Playwright, unit vs integration
│                                      #   vs E2E boundaries, fixture/mock patterns
└── infra/
    └── CLAUDE.md                      # Infrastructure: Cloudflare Workers, Docker,
                                       #   GitHub Actions workflows, deployment
```

---

## Key Files Explained

### `CLAUDE.md` (root)

The single most important file. Every Claude Code session reads this automatically. Contains:

- **Quick reference table** — maps each layer to its path, runtime, and entry point
- **Conventions** — TypeScript strict mode, naming, imports, env var handling, error patterns
- **Database / Cache / Edge rules** — how to use Drizzle, Redis, and Cloudflare Workers
- **Commands** — every `pnpm` script the project uses
- **Sub-Agent Workflow** — when to split into agents, dispatch patterns, parallel vs sequential rules

### `.claude/agents/*.md` — 2026 format

Each file starts with YAML frontmatter that pins model, tools, and triggers:

```yaml
---
name: backend
description: Backend specialist for src/backend/. Owns routes, services, DB, cache.
model: claude-sonnet-4-6
tools: Read, Edit, Write, Bash, Grep, Glob
---
```

The body is the agent's system prompt: scope, pre-flight reading list, responsibilities, and a post-completion checklist.

| Agent | Owns | Pinned to | Why this model |
|-------|------|-----------|----------------|
| `backend` | `src/backend/` | Sonnet 4.6 | Mixed reasoning + code |
| `frontend` | `src/frontend/` | Sonnet 4.6 | Mixed reasoning + code |
| `test` | `tests/` + colocated `*.test.ts` | **Haiku 4.5** | High volume, low novelty |
| `infra` | `infra/` + `.github/workflows/` | Sonnet 4.6 | Cost-sensitive but needs care |

**Why per-agent model pinning matters:** if your main session runs Opus and you spawn 5 subagents in parallel, *every subagent inherits Opus unless told otherwise*. Pinning the test agent to Haiku alone cuts that spawn's cost by ~80% with no quality loss for writing tests. Set `CLAUDE_CODE_SUBAGENT_MODEL` env var to change the default for all unpinned subagents.

### Subfolder `CLAUDE.md` files

Auto-loaded by Claude Code when working in that directory. They provide layer-specific patterns without bloating the root file:

- **`src/backend/CLAUDE.md`** — route→service→DB layering, transaction patterns, cache read-through, "adding a new endpoint" step-by-step
- **`src/frontend/CLAUDE.md`** — Server Components vs Client Components, data fetching, shadcn/ui usage, "adding a new page" step-by-step
- **`tests/CLAUDE.md`** — test stack, colocated unit tests, integration test DB setup, E2E auth shortcuts, common mistakes
- **`infra/CLAUDE.md`** — worker constraints, Docker multi-stage build, CI workflow table, "making infra changes" checklist

---

## How Sub-Agent Dispatch Works

The root `CLAUDE.md` defines three dispatch patterns:

**Feature spanning multiple layers (parallel where possible):**
```
1. Explore agent    → understand current code, find integration points
2. Backend agent    → DB schema, API endpoints, cache logic       \
3. Frontend agent   → UI components, data fetching                 } parallel if API contract agreed
4. Test agent       → unit + integration tests (Haiku, batched)
```

**Bug fix:**
```
1. Explore agent → reproduce, find root cause
2. Fix agent     → implement the fix
3. Test agent    → regression test + run suite
```

**DB migration + cache invalidation (sequential):**
```
1. Backend agent → schema, migration, queries, cache (single agent, sequential)
2. Test agent    → integration tests against test DB
```

---

## Generating Agents Interactively

Don't hand-write agent YAML — let Claude do it for you. Run `/agents` inside a Claude session and it walks you through name, description, model pin, tool list, then writes the file. Claude tends to write sharper trigger descriptions than developers do on a first attempt.

---

## Use This As a Starting Point

Copy the structure into your own project and customize:

1. Replace the fictional tech stack with your actual stack
2. Update conventions to match your team's standards
3. Add/remove agent templates based on your project layers
4. **Pin models by cost vs. quality needs**, not uniformly
5. Adjust the dispatch patterns to fit your typical workflows
