# Kata 03: Variables, Configuration & Statusline (~15–25 min)

## Theory — Fundamentals

Claude Code uses a layered configuration system. Settings can come from multiple sources, with a clear precedence order.

### Settings Precedence (highest to lowest)

1. **CLI arguments** — `claude --permission-mode plan`
2. **Local project settings** — `.claude/settings.local.json` (gitignored, personal)
3. **Project settings** — `.claude/settings.json` (committed, shared with team)
4. **User global settings** — `~/.claude/settings.json` (all your projects)

### CLAUDE.md — Project Memory

CLAUDE.md files provide persistent context that loads automatically every session.

| File | Scope | Shared? |
|------|-------|---------|
| `~/.claude/CLAUDE.md` (or `%USERPROFILE%\.claude\CLAUDE.md`) | All your projects | No |
| `CLAUDE.md` (project root) | This project | Yes (git) |
| `src/CLAUDE.md`, `tests/CLAUDE.md`, etc. | Auto-loaded when Claude works under that path | Yes (git) |
| `CLAUDE.local.md` | This project | No (gitignored) |

All matching files are loaded and concatenated.

### Environment Variables

```json
{
  "env": {
    "NODE_ENV": "development",
    "DEBUG": "app:*"
  }
}
```

Key Claude Code env vars:
- `ANTHROPIC_API_KEY` — API authentication
- `CLAUDE_CODE_MAX_OUTPUT_TOKENS` — Output token limit
- `DISABLE_TELEMETRY` — Opt out

---

## Theory — Novel Layer (2026)

### The Statusline — The Cockpit You Didn't Know You Needed

A `statusLine` config in `.claude/settings.json` runs a script on every turn and shows the output at the bottom of the terminal. It receives session JSON on stdin: model, working dir, git state, **context %, cost, rate limit, tokens in/out**.

Most users never customize it. Doing so is a 10-minute investment that pays off every session.

```json
{
  "statusLine": {
    "type": "command",
    "command": ".claude/statusline.sh"
  }
}
```

A minimal `statusline.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
MODEL=$(echo "$INPUT" | jq -r '.model.id // "?"' | sed 's/claude-//;s/-[0-9].*$//')
DIR=$(echo "$INPUT" | jq -r '.workspace.current_dir' | xargs basename)
CTX=$(echo "$INPUT" | jq -r '.context_window.used_percentage // 0' | xargs printf "%.0f")
COST=$(echo "$INPUT" | jq -r '.cost.total_cost_usd // 0' | xargs printf "$%.2f")
BRANCH=$(git -C "$(echo "$INPUT" | jq -r '.workspace.current_dir')" rev-parse --abbrev-ref HEAD 2>/dev/null || echo "-")
echo "📂 $DIR | 🌿 $BRANCH | 🤖 $MODEL | 💭 ${CTX}% ctx | 💸 $COST"
```

Now you can answer "which model am I on, how close to context limit, how much did this session cost" without leaving the input box.

### Token-Routing Env Vars

| Env var | What it does |
|---------|--------------|
| `CLAUDE_CODE_SUBAGENT_MODEL` | Default model for every unpinned subagent (set to `claude-haiku-4-5-20251001` for cheap batched work) |
| `ANTHROPIC_MODEL` | Main session model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Model used for prompt-type hooks, statusline, internal small tasks |
| `MAX_THINKING_TOKENS` | Hard cap on thinking-token spend per turn |

Set these in `.claude/settings.json` under `env`:

```json
{
  "env": {
    "ANTHROPIC_MODEL": "claude-opus-4-7",
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_SMALL_FAST_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

This is the single highest-leverage config change you can make for cost-conscious teams. Main session reasons hard, subagents and judges work cheap.

### Progressive `CLAUDE.md` — Keep It Lean

The 2026 best practice: keep root `CLAUDE.md` **under 60 lines**. Push detail into:

- `src/<area>/CLAUDE.md` — auto-loaded when working in that path
- Skills with `references/` — loaded only when relevant
- `docs/` files that you point to ("See docs/api-guidelines.md for endpoint patterns")

A 5,000-token root `CLAUDE.md` costs 5,000 tokens **every turn, every session, forever**. A lean root + path-scoped expansion is dramatically cheaper.

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-03/.claude && cd /tmp/kata-03
git init
echo '{ "name": "kata-03" }' > package.json
echo "console.log('hello');" > index.js
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-03\.claude | Out-Null
Set-Location $env:TEMP\kata-03
git init
'{ "name": "kata-03" }' | Out-File -Encoding utf8 package.json
"console.log('hello');" | Out-File -Encoding utf8 index.js
```

### Tasks — Fundamentals

#### 1. Create a Lean CLAUDE.md

Create `/tmp/kata-03/CLAUDE.md` — keep it short:

```markdown
# Kata 03 Project

## Stack
- Node.js, vanilla JS, no frameworks

## Conventions
- camelCase variables, const over let
- JSDoc on exported functions only

## Commands
- Run: `node index.js`
```

Start Claude and ask: `"What conventions does this project use?"` — it should answer from CLAUDE.md without reading other files.

#### 2. Add Personal Overrides

Create `/tmp/kata-03/CLAUDE.local.md`:

```markdown
# My Personal Notes
- Prefer verbose error messages
- Explain reasoning step-by-step
```

Restart Claude and ask: `"How should you communicate with me?"` — both files load.

#### 3. Layer Settings Files

`.claude/settings.json` (shared):
```json
{ "permissions": { "allow": ["Bash(node *)"] } }
```

`.claude/settings.local.json` (personal, gitignored):
```json
{ "permissions": { "allow": ["Bash(node *)", "Bash(git *)"] } }
```

Test that `node index.js` works and `git status` works **only** because of the local file.

#### 4. Use `/init` to Bootstrap a New Project

```bash
mkdir -p /tmp/kata-03-init && cd /tmp/kata-03-init && git init
claude
```

Type `/init` and let Claude generate a CLAUDE.md by inspecting the (empty) project. Review what it produces.

### Tasks — Novel Layer

#### 5. Install a Statusline

Create `/tmp/kata-03/.claude/statusline.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
MODEL=$(echo "$INPUT" | jq -r '.model.id // "?"' | sed 's/claude-//;s/-[0-9].*$//')
DIR=$(echo "$INPUT" | jq -r '.workspace.current_dir' | xargs basename)
CTX=$(echo "$INPUT" | jq -r '.context_window.used_percentage // 0' | xargs printf "%.0f")
COST=$(echo "$INPUT" | jq -r '.cost.total_cost_usd // 0' | xargs printf "$%.2f")
echo "📂 $DIR | 🤖 $MODEL | 💭 ${CTX}% ctx | 💸 $COST"
```

```bash
chmod +x /tmp/kata-03/.claude/statusline.sh
```

Add to `.claude/settings.json`:

```json
{
  "permissions": { "allow": ["Bash(node *)"] },
  "statusLine": {
    "type": "command",
    "command": ".claude/statusline.sh"
  }
}
```

Restart Claude. Look at the bottom of the terminal — you now see your context usage and cost on every turn.

**Windows note:** statusline runs through Git Bash if installed; use forward slashes in the path. Or invoke a PowerShell script: `"command": "powershell -NoProfile -File ./.claude/statusline.ps1"`.

#### 6. Configure Subagent Routing

Add to `.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

Now every subagent your main session spawns will use Haiku by default. Cheap. Test by asking: `"Use a subagent to list every file in this directory and summarize."` Watch the subagent run on Haiku in the trace.

#### 7. (Optional) Install a Polished Statusline

If you want to skip the DIY and just see what a polished statusline looks like:

```bash
# Inside Claude Code:
/plugin marketplace add Owloops/claude-powerline
/plugin install claude-powerline@claude-powerline
/powerline
```

That marketplace plugin ships a configurable vim-style powerline with token usage, cost, rate limits, themes. Good reference for what's possible.

### Discussion Points

- What goes in shared CLAUDE.md vs personal CLAUDE.local.md?
- What's the right size for a root CLAUDE.md? Where would you push detail instead?
- Which env vars would you set for a cost-sensitive team? A latency-sensitive one?
- What three things would *you* put in your statusline?
