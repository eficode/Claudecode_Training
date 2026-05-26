# Kata 01: Agent Modes & Effort Levels (~15–25 min)

## Theory — Fundamentals

Claude Code has **permission modes** that control how much autonomy the agent has, and **effort levels** that control how much it thinks per turn. Different dials, used together.

### Permission Modes

| Mode | What it does | When to use |
|------|-------------|-------------|
| **Default** | Asks permission for edits and commands | Learning, safety-critical work |
| **Auto-accept edits** | Edits files freely, still asks for bash | Refactoring sessions |
| **Plan mode** | Read-only, no code changes allowed | Architecture review, exploration |
| **Bypass permissions** | Skips all checks | Only in isolated containers/CI |

How to switch:
- **During session**: `Shift+Tab` cycles through modes
- **At startup**: `claude --permission-mode plan`
- **In settings**: `"defaultMode"` in `.claude/settings.json`

Plan mode is special: Claude can read, search, and analyze, but **cannot write or execute**. Produces an implementation plan for your approval before any code changes.

---

## Theory — Novel Layer (2026)

### Effort Levels — How Hard Claude Thinks Per Turn

A separate dial from permission mode. Controls the thinking-token budget. Set via the `/effort` slash command:

| Level | Use for |
|-------|---------|
| `low` | Trivial edits, one-line changes, comments |
| `medium` | Normal coding (default) |
| `high` | Architecture, tricky bugs, refactors that span files |
| `xhigh` / `max` | Hard reasoning problems (security review, perf analysis), Opus only |
| `auto` | Let Claude pick per turn |

Why this matters: `max` thinking can burn tens of thousands of output tokens per turn. Running it on a typo fix is pure waste. Running `low` on a security review is dangerous. Match the dial to the task.

### Background Mode

Append `&` to a prompt and that turn runs as a background agent — your terminal stays responsive while Claude works. Useful for long-running tasks like "run the full test suite and fix failures."

### Agent View — The Dashboard

`claude agents` (no other args) opens a full-screen TUI showing every background and resumable session: what's running, what needs input, what's done. You can dispatch new sessions from this view, peek at progress without interrupting, and attach when you need the full conversation. Sessions survive terminal closure — a supervisor process keeps them alive.

### Headless Mode (`claude -p`)

`claude -p "your prompt"` runs Claude non-interactively — useful for CI, cron jobs, scripts. Output to stdout, exit code reflects success. Starting 2026-06-15, headless subprocess calls draw from a separate **Agent SDK credit pool**, so they don't compete with your interactive quota.

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-01 && cd /tmp/kata-01
git init
echo "console.log('hello');" > app.js
echo '{ "name": "kata-01", "scripts": { "start": "node app.js" } }' > package.json
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-01 | Set-Location
git init
"console.log('hello');" | Out-File -Encoding utf8 app.js
'{ "name": "kata-01", "scripts": { "start": "node app.js" } }' | Out-File -Encoding utf8 package.json
```

### Tasks — Fundamentals

#### 1. Start Claude Code in Plan Mode

```bash
cd /tmp/kata-01
claude --permission-mode plan
```

Ask Claude: `"Analyze app.js and suggest how to add Express server functionality. Don't change anything, just plan."`

Observe that Claude reads files but never writes. Press `Ctrl+C` to exit.

#### 2. Cycle Modes with Shift+Tab

```bash
claude
```

Press `Shift+Tab` repeatedly — watch the mode indicator change in the status bar. Note that auto-accept skips the approval dialog for edits.

#### 3. Compare Default vs Auto-Accept

In **default mode**, ask: `"Add a comment to the top of app.js."` Approve or deny.
In **acceptEdits mode** (`claude --permission-mode acceptEdits`), ask the same thing. Claude edits without asking.

### Tasks — Novel Layer

#### 4. Try Effort Levels

Inside a Claude session:

```
/effort low
Add a // TODO comment to app.js
```

Then:

```
/effort high
Refactor app.js into an Express server with /health and /api/users endpoints. Use plan mode first.
```

Observe: the low-effort turn returns almost instantly with minimal "thinking." The high-effort turn takes longer, shows more reasoning, and produces a more considered answer.

#### 5. Dispatch a Background Agent

In a Claude session, append `&` to a prompt:

```
Read every file in this project and produce a one-page summary of what it does &
```

Notice your prompt returns to you immediately. The background agent runs while you do other things.

#### 6. Open the Agent View Dashboard

In a separate terminal:

```bash
claude agents
```

You should see your background agent from step 5. Peek at progress, then attach with Enter when you want to see the full conversation.

#### 7. Try Headless Mode

```bash
claude -p "List the public functions in app.js. Output JSON only."
```

Notice you get just the output, no TUI. This is what your CI pipeline runs.

### Discussion Points

- When would you use Plan mode in your daily workflow?
- Which effort level fits your usual workload? Where would `xhigh` actually pay off?
- What would you automate with `claude -p` in CI?
- What's the risk of `acceptEdits` for a teammate who isn't paying attention?
