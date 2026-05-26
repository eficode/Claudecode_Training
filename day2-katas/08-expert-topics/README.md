# Kata 08: Expert Topics — Orchestration, Marketplaces & Headless Workflows (Reference)

> Reference guide, not a hands-on kata. Covers advanced Claude Code internals and ecosystem patterns for senior audiences.

---

## Topic A: Subagents, Agent Teams & Per-Model Routing

### What Are Subagents?

Claude Code can spawn **subagents** — independent Claude instances that run in isolated context windows. The parent agent delegates work and receives results without polluting its own context.

```
User Prompt
    |
    v
Main Agent (e.g., Opus 4.7)
    |-- Task: explore   → Explore subagent (Sonnet)  ─┐
    |-- Task: bash      → Bash subagent (Haiku)       │ parallel
    |-- Task: review    → Review subagent (Opus, xhigh)┘
    v
Main Agent synthesizes results
```

### Built-in Subagent Types

| Type | Tools Available | Use Case |
|------|----------------|----------|
| **Explore** | Read, Glob, Grep, WebFetch, WebSearch | Codebase exploration, research |
| **Bash** | Bash only | Command execution, git ops |
| **Plan** | Read, Glob, Grep (no writes) | Architecture planning |
| **general-purpose** | All tools | Complex multi-step tasks |

### Per-Agent Model Selection (2026)

Each subagent can be pinned to a specific model in its YAML frontmatter:

```yaml
---
name: code-reviewer
description: Review code changes for quality and security issues
model: claude-opus-4-7
tools: Read, Grep, Glob
---
```

Pattern: main session runs Opus (expensive thinking), heavy reasoning agents run Opus, high-volume agents (tests, doc updates) run Haiku. Cut your subagent costs ~5–10× with no quality loss on the right tasks.

Set `CLAUDE_CODE_SUBAGENT_MODEL=claude-haiku-4-5-20251001` to change the default for every unpinned subagent.

### Agent View (`claude agents`)

Full-screen TUI showing every background and resumable session. Dispatch new sessions, peek at progress without attaching, attach to take over. Sessions survive terminal closure — a supervisor keeps them alive.

### Agent Teams (Experimental)

Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Now an orchestrator can dispatch worker agents that **message each other and share findings**. Use sparingly — every inter-agent message is a billable round trip. Right pattern when work is interdependent and parallel subagents (which can't talk) aren't enough.

### Subagent Patterns

**Pattern 1 — Parallel Research:**
```
Main: "Refactor the auth system"
  |-- Explore: find all auth files
  |-- Explore: find session management
  |-- Explore: find middleware patterns
  v (all 3 return in parallel)
Main: synthesize, plan
```

**Pattern 2 — Background Workers:**
```
Main: editing code
  |-- Bash (background): npm run test:watch
  |-- Bash (background): npm run build
Main: continues, checks results later
```

**Pattern 3 — Worktree Isolation:**
```
git worktree add ../feature-a feature-a
git worktree add ../feature-b feature-b
# Three terminals: main session in original, and one Claude per worktree
# Each agent works on a different feature branch; no merge conflicts
```

### Skills as Subagents

Skills with `context: fork` in their frontmatter run as subagents:

```yaml
---
name: deep-review
context: fork
allowed-tools: Read, Grep, Glob
---

Perform an exhaustive code review of $ARGUMENTS...
```

Keeps the review's large context isolated from the main conversation.

---

## Topic B: Plugin Marketplaces (Public & Private)

### State of the Ecosystem (May 2026)

- **claude-plugins-official** (Anthropic-managed): 101+ plugins, auto-available in every Claude Code install
- **75+ community marketplaces**: ~1,200 plugins total
- Notable partners: GitHub, Playwright, Supabase, Figma, Vercel, Linear, Sentry, Stripe, Firebase

Browse at `claude.com/plugins` or via `/plugin` → Discover tab.

### Distribution Patterns for Private Plugins

#### 1. Git-Based Marketplace (Simplest)

A git repo with a `marketplace.json` manifest:

```
acme-claude-plugins/
└── .claude-plugin/
    └── marketplace.json
```

```json
{
  "name": "acme-internal",
  "version": "1.0.0",
  "plugins": [
    { "name": "acme-deploy", "source": "./plugins/acme-deploy" },
    { "name": "acme-review", "source": "./plugins/acme-review" }
  ]
}
```

Developers:

```bash
/plugin marketplace add yourco/acme-claude-plugins
/plugin install acme-deploy@acme-internal
```

PR-based governance. Git tags = versions. `git revert` = rollback.

#### 2. NPM-Based for MCP Servers

For MCP servers, scoped npm packages on a private registry (Verdaccio, GitHub Packages, Artifactory):

```bash
npm install --save-dev @yourco/claude-mcp-jira
```

Reference in `.mcp.json`:

```json
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["@yourco/claude-mcp-jira"],
      "env": { "JIRA_TOKEN": "${JIRA_TOKEN}" }
    }
  }
}
```

#### 3. OCI/Container Distribution

For MCP servers that need isolated environments:

```json
{
  "mcpServers": {
    "secure-db": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "registry.yourco.com/claude-mcp-db:latest"]
    }
  }
}
```

### Governance Considerations

| Concern | Solution |
|---------|----------|
| **Versioning** | Pin plugin versions; git tags or npm versions |
| **Approval** | PR-based plugin submissions with security review |
| **Auditing** | HTTP hooks → central log; MCP servers can log internally |
| **Rollback** | `git revert` or pin to prior version |
| **Access control** | Private repos/registries with team-scoped permissions |
| **Supply-chain** | Audit skills + MCP configs before merging to internal marketplace |

---

## Topic C: Headless Workflows & The Claude Code SDK

### Headless Mode (`claude -p`)

Run Claude Code non-interactively:

```bash
claude -p "Summarize the changes in this PR" \
  --permission-mode acceptEdits \
  --max-turns 5
```

Output to stdout. Exit code reflects success/failure. **As of 2026-06-15, headless calls draw from a separate Agent SDK credit pool** — they don't compete with your interactive quota. Good for CI, cron, scripts.

### Remote Control (Q1 2026)

Claude Code can run as a persistent process you trigger and query via API/webhook. One orchestrator service dispatches tasks to multiple Claude Code instances on different machines. Enables:

- Scheduled autonomous tasks on cron
- CI/CD pipeline steps that open PRs, run tests, fix failures
- Phone-based monitoring of long-running background agents

### Claude Code SDK

The `@anthropic-ai/claude-code` SDK lets you build custom agent workflows programmatically:

```bash
npm install @anthropic-ai/claude-code
```

```typescript
import { query } from "@anthropic-ai/claude-code";

// Single query
const result = await query({
  prompt: "Analyze the auth system and suggest improvements",
  options: {
    maxTurns: 10,
    model: "claude-sonnet-4-6",
    permissionMode: "plan",
  },
});

// Chained workflows
const analysis = await query({
  prompt: "Find all security vulnerabilities",
  options: { maxTurns: 5, model: "claude-opus-4-7" },
});

const fix = await query({
  prompt: `Fix these issues: ${analysis.result}`,
  options: { maxTurns: 15, model: "claude-sonnet-4-6" },
});
```

### CI Patterns That Work

| Pattern | Example |
|---------|---------|
| **Auto-review on PR open** | GitHub Action → `claude -p "Review this diff for security and style"` → posts comment |
| **Auto-fix on test fail** | CI runs tests → on red, `claude -p "Fix the failing tests"` → opens patch PR |
| **Doc sync** | Nightly cron → `claude -p "Update README to reflect current public API"` |
| **Migration generator** | On model change → `claude -p "Write a Drizzle migration from old schema to new"` |

---

## Topic D: Token Economics & Context Engineering

### The Three Token Drains

1. **CLAUDE.md** — every line costs tokens every turn. A 5,000-token root file = 5,000 tokens *minimum* per turn.
2. **Auto-read files** — Claude slurps files when globbing. `.claudeignore` is critical.
3. **MCP tool responses** — full JSON joins the context. Summarize on the server side when possible.

### Mitigation Stack

| Lever | Impact |
|-------|--------|
| Lean root `CLAUDE.md` (<60 lines) + path-scoped sub-CLAUDE.mds | High |
| `.claudeignore` for `node_modules`, `dist`, lockfiles, `.env*` | High |
| Subagents for heavy reads — return summaries, not raw content | High |
| `CLAUDE_CODE_SUBAGENT_MODEL=haiku` for unpinned subagents | High |
| Progressive-disclosure skills (refs loaded only when needed) | Medium |
| Capping `bash` output: `--max-output 20000` or `head -n` filters | Medium |
| `/handoff` at ~70% context instead of letting it rot to 100% | Medium |
| `/effort low` for trivial turns | Low–Medium |

### Bedrock & Cross-Region Notes

For ACME-corp-style AWS Bedrock deployments, model routing happens via `ANTHROPIC_MODEL` in settings:

```json
{
  "env": {
    "ANTHROPIC_MODEL": "global.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "ANTHROPIC_SMALL_FAST_MODEL": "eu.anthropic.claude-haiku-4-5-20251001-v1:0"
  }
}
```

If a session feels slow, the culprit is usually account-level throttling on the chosen model in your Bedrock account — switch via env var and observe. EU and Global cross-region models are typically available.

---

## Further Reading

- [Claude Code documentation](https://docs.claude.com/en/docs/claude-code)
- [Model Context Protocol specification](https://modelcontextprotocol.io/)
- [Official plugin marketplace catalog](https://claude.com/plugins)
- [Claude Code SDK on npm](https://www.npmjs.com/package/@anthropic-ai/claude-code)
- [Anthropic Engineering: Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
