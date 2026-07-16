# Kata 0: Directory & Config Files Example (Reference)

## Overview

Reference blueprint for a Claude Code project in 2026. Shows how `CLAUDE.md`, `.claude/`, plugins, skills, hooks, MCP servers, statusline, and per-agent model routing fit together.

Use this as a starting template when setting up Claude Code on a real codebase.

---

## Directory Structure (2026)

```
my-saas-app/
├── CLAUDE.md                              # Root project memory
├── CLAUDE.local.md                        # Personal overrides (gitignored)
├── .mcp.json                              # Project-scoped MCP servers
│
├── .claude/
│   ├── settings.json                      # Project settings (shared via git)
│   ├── settings.local.json                # Local overrides (gitignored)
│   ├── statusline.sh                      # Custom status bar (model, context %, cost)
│   │
│   ├── agents/                            # Subagent configs (one YAML per specialist)
│   │   ├── backend.md                     #   pinned to claude-sonnet-4-6
│   │   ├── frontend.md
│   │   ├── reviewer.md                    #   pinned to claude-opus-4-7 + xhigh effort
│   │   └── janitor.md                     #   pinned to claude-haiku-4-5 (cheap)
│   │
│   ├── rules/
│   │   ├── code-style.md                  # Unconditional style rules
│   │   ├── api-guidelines.md              # Path-scoped API rules
│   │   ├── testing.md
│   │   └── security.md
│   │
│   ├── skills/                            # Custom slash commands + skill bundles
│   │   ├── deploy/
│   │   │   ├── SKILL.md                   #   /deploy — lean entry point
│   │   │   ├── references/                #     loaded only when needed
│   │   │   │   ├── aws.md
│   │   │   │   └── gcp.md
│   │   │   └── scripts/
│   │   │       └── precheck.sh            #     deterministic helper
│   │   ├── db-migrate/SKILL.md
│   │   └── review-security/SKILL.md
│   │
│   ├── plugins/                           # Installed via /plugin install
│   │   └── (auto-managed)
│   │
│   └── hooks/                             # Lifecycle automation
│       ├── pre-commit-lint.sh             # PostToolUse(Edit|Write)
│       ├── validate-migrations.sh         # PreToolUse(Bash) — block bad migrations
│       ├── check-secrets.sh               # PreToolUse(Edit|Write) — block secret leaks
│       ├── inject-todos.sh                # SessionStart — load today's todos
│       └── backup-state.js                # PreCompact — save context before compaction
│
├── scripts/
│   ├── mcp/
│   │   ├── package.json                   # MCP server deps
│   │   └── project-context-server.ts      # Custom MCP server (with elicitation)
│   └── ci/
│       └── claude-review.sh               # Headless Claude in CI (claude -p)
│
├── src/
│   ├── CLAUDE.md                          # Auto-loaded when working under src/
│   ├── api/
│   │   └── CLAUDE.md                      # Auto-loaded under src/api/
│   ├── components/
│   └── lib/
│
├── docs/
│   ├── architecture.md
│   ├── onboarding.md
│   └── specs/                             # Spec-driven development outputs
│       └── 2026-05-pricing-v2.md
│
├── .gitignore                             # includes CLAUDE.local.md, .claude/settings.local.json
└── README.md
```

---

## How It All Connects

| Feature | File(s) | Purpose |
|---------|---------|---------|
| **Project memory** | `CLAUDE.md` | Commands, conventions, architecture |
| **Personal overrides** | `CLAUDE.local.md` | Your local env quirks (gitignored) |
| **Subfolder memory** | `src/CLAUDE.md`, `src/api/CLAUDE.md` | Auto-loaded when Claude touches those dirs |
| **Modular rules** | `.claude/rules/*.md` | Path-scoped style/security/testing rules |
| **Permissions** | `.claude/settings.json` | Allowlist/denylist for tool use |
| **Hooks** | `.claude/settings.json` → `.claude/hooks/` | Lifecycle automation across 27+ events |
| **Statusline** | `.claude/settings.json` → `.claude/statusline.sh` | Live context %, cost, model, branch |
| **Skills** | `.claude/skills/*/SKILL.md` | `/deploy`, `/db-migrate`, `/review-security` |
| **Subagents** | `.claude/agents/*.md` | Per-specialist YAML + model pinning |
| **Plugins** | `.claude/plugins/` (auto) | Marketplace installs: `/plugin install <name>` |
| **MCP servers** | `.mcp.json` → `scripts/mcp/` | DB introspection, Linear, Sentry, internal APIs |
| **CI integration** | `scripts/ci/claude-review.sh` | Headless `claude -p` in GitHub Actions |

---

## Minimal `.claude/settings.json`

```json
{
  "statusLine": {
    "type": "command",
    "command": ".claude/statusline.sh"
  },
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5-20251001"
  },
  "permissions": {
    "allow": ["Bash(pnpm *)", "Bash(git status)", "Bash(git diff)"],
    "deny": ["Bash(rm -rf *)", "Read(.env*)"]
  },
  "hooks": {
    "PreToolUse": [
      { "matcher": "Edit|Write", "hooks": [
        { "type": "command", "command": ".claude/hooks/check-secrets.sh" }
      ]}
    ],
    "PostToolUse": [
      { "matcher": "Edit|Write", "hooks": [
        { "type": "command", "command": ".claude/hooks/pre-commit-lint.sh", "async": true }
      ]}
    ],
    "SessionStart": [
      { "hooks": [
        { "type": "command", "command": ".claude/hooks/inject-todos.sh" }
      ]}
    ]
  }
}
```

The `env.CLAUDE_CODE_SUBAGENT_MODEL` line is the cheapest token win in the entire stack — it routes every subagent to Haiku by default while your main session stays on Sonnet/Opus.
