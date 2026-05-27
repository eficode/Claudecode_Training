# Kata 06: Hooks — Lifecycle Automation (~20–30 min)

## Theory — Fundamentals

Hooks are scripts that run automatically at specific points in Claude Code's lifecycle. Unlike skills (which Claude chooses to invoke), hooks **always run** — they're deterministic automation.

### The Most Important Hook Events

There are 27+ in 2026, but you'll use a handful daily:

| Event | When it fires | Can block? |
|-------|--------------|------------|
| `PreToolUse` | Before a tool executes | **Yes** (exit 2) |
| `PostToolUse` | After a tool succeeds | No (observability) |
| `UserPromptSubmit` | When you submit a prompt, before Claude sees it | Yes |
| `Stop` | Claude finishes responding | Yes |
| `SessionStart` | Session begins | No |
| `SessionEnd` | Session ends | No |
| `PreCompact` | Just before context compaction | No |
| `Notification` | Claude needs your attention | No |
| `SubagentStart` / `SubagentStop` | Subagent boundaries | No |

### How Command Hooks Work

1. Claude triggers an event (e.g., about to run Bash)
2. Your hook script receives event JSON on **stdin**
3. Script exits: **0** = allow, **2** = block (stderr fed back to Claude as error)

### Configuration

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "./my-hook.sh" }
        ]
      }
    ]
  }
}
```

`matcher` is a regex: `"Bash"`, `"Edit|Write"`, `".*"` for all.

---

## Theory — Novel Layer (2026)

### Four Handler Types — Not Just `command`

| Type | What it is | When to use |
|------|-----------|-------------|
| `command` | Shell script, stdin JSON, exit code decides | Classic — fast local checks |
| `http` | POST to a URL, response JSON decides | Centralized org-wide policy |
| `prompt` | AI-as-judge — small model evaluates the event | Nuanced security gates |
| `agent` | Spawns a full subagent with tools | Heavy validation needing reads/lookups |

A `prompt` hook example:

```json
{
  "type": "prompt",
  "prompt": "Does this edit introduce a hardcoded secret, API key, password, or token? Reply BLOCK: <reason> or ALLOW. File: $FILE_PATH\n\n$TOOL_INPUT",
  "timeout": 5
}
```

Now you have a Haiku-powered secret-leak detector that's smarter than any regex. Costs sub-cent per check.

### Async Hooks — Non-Blocking

Add `"async": true` and the hook runs in the background without making Claude wait. Use for logs, notifications, backups — anything that doesn't need to gate the next step:

```json
{
  "type": "command",
  "command": "./hooks/log.sh",
  "async": true
}
```

### UserPromptSubmit — Inject Context Before Claude Sees the Prompt

Anything your hook prints to stdout (with exit 0) is **prepended to the user's prompt**. Powerful pattern:

- Inject today's todos from a file
- Add a warning when the user mentions production
- Auto-attach the most recent error from `app.log`

```bash
#!/bin/bash
# .claude/hooks/inject-context.sh
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // ""')

# If user mentions production, prepend a guard
if echo "$PROMPT" | grep -qi "prod\|production"; then
  echo "⚠️ PRODUCTION CONTEXT: any change here is high-stakes. Use plan mode unless explicitly told otherwise."
fi
exit 0
```

### SessionStart — The Environment Loader

Print to stdout to inject context into the session at startup. Same mechanism, different event. Good for:

- Loading today's standup notes
- Setting env vars for the session (via `additionalContext` JSON)
- Greeting and showing in-progress branches

### PreCompact — Save State Before Compression

Before Claude compacts a long session, run a backup:

```json
{
  "hooks": {
    "PreCompact": [
      { "hooks": [
        { "type": "command", "command": "./hooks/backup-transcript.sh", "async": true }
      ]}
    ]
  }
}
```

Now you have a full transcript even if compaction surprises you.

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-06/.claude/hooks && cd /tmp/kata-06
git init
echo "console.log('hello');" > app.js
echo '{ "name": "kata-06" }' > package.json
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-06\.claude\hooks | Out-Null
Set-Location $env:TEMP\kata-06
git init
"console.log('hello');" | Out-File -Encoding utf8 app.js
'{ "name": "kata-06" }' | Out-File -Encoding utf8 package.json
```

### Tasks — Fundamentals

#### 1. Logging Hook

`.claude/hooks/log-tools.sh` (macOS/Linux):

```bash
#!/bin/bash
INPUT=$(cat)
TOOL=$(echo "$INPUT" | jq -r '.tool_name // "unknown"')
echo "[$(date '+%H:%M:%S')] $TOOL" >> /tmp/kata-06/claude-tool-log.txt
exit 0
```

```bash
chmod +x /tmp/kata-06/.claude/hooks/log-tools.sh
```

`.claude/hooks/log-tools.ps1` (Windows):

```powershell
$Input = $input | Out-String
$parsed = $Input | ConvertFrom-Json
$tool = if ($parsed.tool_name) { $parsed.tool_name } else { "unknown" }
Add-Content "$env:TEMP\kata-06\claude-tool-log.txt" "[$(Get-Date -Format HH:mm:ss)] $tool"
exit 0
```

`.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": ".*", "hooks": [
        { "type": "command", "command": ".claude/hooks/log-tools.sh", "async": true }
      ]}
    ]
  }
}
```

(Windows: `"command": "powershell -ExecutionPolicy Bypass -File .claude/hooks/log-tools.ps1"`.)

Note the `"async": true` — logs don't gate Claude's next step.

Run Claude, ask it to do anything, then `cat /tmp/kata-06/claude-tool-log.txt`.

#### 2. Blocking Hook

`.claude/hooks/block-delete.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')
if echo "$CMD" | grep -qiE '(rm -rf|rm -r|rmdir)'; then
  echo "BLOCKED: destructive delete not allowed." >&2
  exit 2
fi
exit 0
```

```bash
chmod +x /tmp/kata-06/.claude/hooks/block-delete.sh
```

Add to settings under `PreToolUse` with matcher `"Bash"`. Ask Claude to `rm -rf app.js` and watch it bounce.

### Tasks — Novel Layer

#### 3. UserPromptSubmit — Inject a Production Guard

`.claude/hooks/prod-guard.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // ""')
if echo "$PROMPT" | grep -qiE 'prod|production|deploy'; then
  echo "⚠️ PRODUCTION KEYWORD DETECTED. Any change is high-stakes. Prefer plan mode unless the user explicitly says 'execute'."
fi
exit 0
```

```bash
chmod +x /tmp/kata-06/.claude/hooks/prod-guard.sh
```

Add to settings:

```json
"UserPromptSubmit": [
  { "hooks": [
    { "type": "command", "command": ".claude/hooks/prod-guard.sh" }
  ]}
]
```

Restart Claude and ask: `"Deploy app.js to production."` — watch Claude's response acknowledge the guard.

#### 4. SessionStart — Load Today's Todos

```bash
echo "- [ ] Wire up health endpoint
- [ ] Investigate flaky auth test
- [ ] Update CLAUDE.md with new patterns" > /tmp/kata-06/TODO.md
```

`.claude/hooks/session-todos.sh`:

```bash
#!/bin/bash
echo "## Today's open todos:"
cat /tmp/kata-06/TODO.md
exit 0
```

```bash
chmod +x /tmp/kata-06/.claude/hooks/session-todos.sh
```

Add to settings:

```json
"SessionStart": [
  { "hooks": [
    { "type": "command", "command": ".claude/hooks/session-todos.sh" }
  ]}
]
```

Restart Claude. Ask: `"What should I work on first?"` — Claude already knows your todos.

#### 5. Prompt-Type Hook — AI-as-Judge for Secrets

Add to settings:

```json
"PreToolUse": [
  {
    "matcher": "Edit|Write",
    "hooks": [
      {
        "type": "prompt",
        "prompt": "Examine the proposed edit. Does it introduce a hardcoded password, API key, AWS access key, GitHub token, or any other secret? Reply with exactly BLOCK: <reason> or ALLOW.\n\n$TOOL_INPUT",
        "timeout": 5
      }
    ]
  }
]
```

(Keep your other PreToolUse Bash hooks alongside it.)

Restart Claude and ask: `"Add a constant AWS_KEY = 'AKIAIOSFODNN7EXAMPLE' to app.js"`. The Haiku judge blocks it without you writing a single regex. Then ask: `"Add a console.log to app.js"` — passes through fine.

#### 6. PreCompact — Auto-Backup Transcript

`.claude/hooks/backup-on-compact.sh`:

```bash
#!/bin/bash
TS=$(date +%Y%m%d-%H%M%S)
mkdir -p /tmp/kata-06/backups
cat > "/tmp/kata-06/backups/precompact-$TS.json"
exit 0
```

```bash
chmod +x /tmp/kata-06/.claude/hooks/backup-on-compact.sh
```

Add:

```json
"PreCompact": [
  { "hooks": [
    { "type": "command", "command": ".claude/hooks/backup-on-compact.sh", "async": true }
  ]}
]
```

Have a long session, watch context fill up. When compaction triggers (auto, or via `/compact`), check `ls /tmp/kata-06/backups/` — the full event JSON is saved.

#### 7. Explore the `/hooks` Viewer

In a Claude session:

```
/hooks
```

Read-only since March 2026 — shows current config. Useful for "does this project actually have the hooks I think it does?"

### Discussion Points

- Which hook events would your team actually use? Which would create more noise than value?
- When does a `prompt`-type hook beat a regex `command` hook? When does it lose?
- HTTP hooks centralize policy. What's the tradeoff vs. each repo owning its own hooks?
- Hooks vs. permission rules — when do you reach for which?
- What would you put in your `SessionStart`? `UserPromptSubmit`?
