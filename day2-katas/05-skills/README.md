# Kata 05: Skills, Progressive Disclosure & Plugin Marketplaces (~20–30 min)

## Theory — Fundamentals

Skills are reusable prompt templates that extend Claude Code with custom slash commands. They package workflows, coding standards, and automation into shareable commands.

### Where Skills Live

| Location | Scope | Shared? |
|----------|-------|---------|
| `~/.claude/skills/<name>/SKILL.md` (or `%USERPROFILE%\.claude\skills\<name>\SKILL.md`) | All your projects | No |
| `.claude/skills/<name>/SKILL.md` | This project | Yes (git) |

### Minimal SKILL.md

```yaml
---
name: my-skill
description: What this skill does and when to invoke it
allowed-tools: Read, Grep, Bash(npm *)
argument-hint: [filename]
---

Your instructions to Claude here.

Use $ARGUMENTS to reference what the user passes.
```

### Key Frontmatter Options

| Option | Purpose |
|--------|---------|
| `name` | Becomes the `/command-name` |
| `description` | **Trigger** for auto-invocation — write it for the model, not for humans |
| `disable-model-invocation: true` | Only manual `/name` trigger; Claude won't auto-use |
| `allowed-tools` | Restrict which tools the skill can use |
| `argument-hint` | autocomplete placeholder showing what input the user should provide |
| `context: fork` | Run in a subagent (separate context) |

### Dynamic Content

Inject command output with `!` backticks — Claude only sees the *result*, not the command:

```markdown
Current branch: !`git rev-parse --abbrev-ref HEAD`
Recent changes: !`git log --oneline -5`
```

---

## Theory — Novel Layer (2026)

### Progressive Disclosure — Skills Are Folders, Not Files

The 2024-style "one big SKILL.md" is over. In 2026, well-built skills are *folders* with structured sub-paths. Claude loads `SKILL.md` first; it points to `references/`, `scripts/`, and `examples/` that Claude reads **only when relevant**.

```
.claude/skills/deploy/
├── SKILL.md              ← lean (under 500 lines): overview + when to read what
├── references/
│   ├── aws.md            ← only read if deploying to AWS
│   ├── gcp.md            ← only read if deploying to GCP
│   └── rollback.md       ← only read on rollback
├── scripts/
│   ├── precheck.sh       ← deterministic helper Claude executes, not reads
│   └── notify.sh
└── examples/
    └── canary-deploy.md  ← few-shot examples
```

Why this matters:
- **Token efficiency**: a 5,000-line skill that loads entirely on every invocation is dead weight. A 200-line SKILL.md + 4,800 lines of references loaded selectively is the same knowledge for a fraction of the context cost.
- **Reliability**: deterministic scripts (`scripts/precheck.sh`) execute identically every time. Letting Claude *re-derive* the same logic in prose is where bugs live.

### Anatomy of a Good SKILL.md (2026)

```yaml
---
name: deploy
description: Use this skill when the user wants to deploy, rollback, or check deployment status. Handles AWS and GCP. Performs preflight checks and emits a deploy summary.
allowed-tools: Read, Bash, Grep
argument-hint: [environment]
---

# Deploy

## Quick start
1. Run `scripts/precheck.sh` — fails fast if branch is dirty or tests are red.
2. Determine the target cloud (AWS vs GCP) and read the matching reference:
   - AWS → see `references/aws.md`
   - GCP → see `references/gcp.md`
3. Execute deploy via the provider-specific commands in that reference.
4. Run `scripts/notify.sh` to post the deploy summary to Slack.

## Gotchas
- If `precheck.sh` exits non-zero, **do not proceed**. Report what failed.
- Never deploy from a non-main branch without confirming with the user.

## Validation
Run `curl -fsS $DEPLOY_URL/health` after deploy. Expect 200 with `{"status":"ok"}`.
```

Notice: the SKILL.md doesn't *contain* the AWS or GCP details. It's a table of contents. Claude reads the right reference only when needed.

### Plugins — Skills As a Distribution Format

In late 2025, Anthropic added **plugins**: a packaging format that bundles skills + MCP servers + hooks + commands into one installable unit. By May 2026 there are 101+ plugins in the official marketplace and ~1,200 across 75 community marketplaces.

```bash
# inside Claude Code
/plugin                                          # opens the plugin browser
/plugin marketplace add owner/repo               # add a community marketplace
/plugin install security-guidance@claude-plugins-official
```

Plugins live in `~/.claude/plugins/` and apply across projects once installed globally. The official Anthropic marketplace ships built-in.

Categories worth knowing: **code-intelligence** (LSP integration — jump-to-def, find-references), **dev workflow** (feature-dev, code-review, commit-commands), **output styles**, partner integrations (GitHub, Playwright, Supabase, Vercel, Linear, Sentry, Stripe).

### Building Your Own Private Marketplace

For sharing skills inside your company without publishing publicly: a marketplace is just a git repo with a manifest. Minimum:

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
    {
      "name": "acme-deploy",
      "description": "Internal deployment workflow",
      "source": "./plugins/acme-deploy"
    }
  ]
}
```

Then each developer:

```bash
/plugin marketplace add yourco/acme-claude-plugins
/plugin install acme-deploy@acme-internal
```

A pull request gates every plugin update. Versioning is git tags. Rollback is `git revert`.

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-05/src && cd /tmp/kata-05
git init
cat > src/app.js << 'JSEOF'
function calculateTotal(items) {
  let total = 0;
  for (let i = 0; i < items.length; i++) {
    total = total + items[i].price * items[i].quantity;
  }
  return total;
}
module.exports = { calculateTotal };
JSEOF
echo '{ "name": "kata-05" }' > package.json
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-05\src | Out-Null
Set-Location $env:TEMP\kata-05
git init
@"
function calculateTotal(items) {
  let total = 0;
  for (let i = 0; i < items.length; i++) {
    total = total + items[i].price * items[i].quantity;
  }
  return total;
}
module.exports = { calculateTotal };
"@ | Out-File -Encoding utf8 src\app.js
'{ "name": "kata-05" }' | Out-File -Encoding utf8 package.json
```

### Tasks — Fundamentals

#### 1. Create a Simple Skill

Create `.claude/skills/review/SKILL.md`:

```yaml
---
name: review
description: Review code for quality issues. Use when the user asks to review, audit, or critique a file.
allowed-tools: Read, Grep, Glob
argument-hint: [file-path]
---

Review the file at $ARGUMENTS for:

1. **Bugs** — logic errors, edge cases, off-by-one
2. **Performance** — unnecessary loops, memory leaks
3. **Readability** — naming, structure, complexity
4. **Modern JS** — could use map/reduce/arrow functions?

Format as a numbered list with severity (LOW/MEDIUM/HIGH).
```

Test:

```bash
cd /tmp/kata-05
claude
```

```
/review src/app.js
```

#### 2. Skill With Dynamic Content

`.claude/skills/status/SKILL.md`:

```yaml
---
name: status
description: Show project status with git context
disable-model-invocation: true
---

## Current Project Status

Git branch: !`git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "no git"`
Recent commits: !`git log --oneline -3 2>/dev/null || echo "no commits"`
Changed files: !`git status --short 2>/dev/null || echo "clean"`

Summarize what's happening and suggest what to work on next.
```

Test with `/status`.

### Tasks — Novel Layer

#### 3. Convert a Skill to Progressive Disclosure

Replace `.claude/skills/review/SKILL.md` with a folder-shaped skill.

```bash
rm /tmp/kata-05/.claude/skills/review/SKILL.md
mkdir -p /tmp/kata-05/.claude/skills/review/references
mkdir -p /tmp/kata-05/.claude/skills/review/scripts
```

`.claude/skills/review/SKILL.md` (lean — under 500 lines):

```yaml
---
name: review
description: Review code for quality issues. Use when the user asks to review, audit, or critique a file.
allowed-tools: Read, Grep, Glob, Bash
argument-hint: [file-path]
---

# Review

## Quick start
1. Read the target file: $ARGUMENTS
2. Identify the language by extension.
3. Read the matching reference checklist:
   - `.js`/`.ts` → see `references/javascript.md`
   - `.py` → see `references/python.md`
   - other → use the general principles in `references/general.md`
4. Run `scripts/complexity.sh $ARGUMENTS` and include the score in your output.
5. Produce a numbered list with severity (LOW/MEDIUM/HIGH).

## Output format
Each finding: severity, line number, one-sentence problem, one-sentence fix.
```

`.claude/skills/review/references/javascript.md`:

```markdown
# JavaScript/TypeScript Review Checklist

- Null/undefined handling on optional fields
- `for (let i = 0...)` patterns where `map`/`reduce`/`forEach` would read clearer
- Mutation of arguments
- Mixing async/await and `.then()` in the same function
- Missing error handling on Promise chains
- Equality with `==` instead of `===`
- Floating-point arithmetic on money (cents, not dollars)
```

`.claude/skills/review/references/python.md`:

```markdown
# Python Review Checklist

- Mutable default args (`def f(x=[]):`)
- `except:` without a specific exception class
- Using `len(x) == 0` instead of `not x`
- Missing type hints on public functions
```

`.claude/skills/review/references/general.md`:

```markdown
# General Review Checklist

- Function does more than one thing
- Names that don't reflect behavior
- Magic numbers without explanation
- Duplicated logic across files
```

`.claude/skills/review/scripts/complexity.sh`:

```bash
#!/bin/bash
# Trivial complexity score: count if/for/while + function depth.
FILE="$1"
[ -z "$FILE" ] && { echo "usage: complexity.sh <file>"; exit 1; }
N=$(grep -cE '\b(if|for|while|switch)\b' "$FILE")
echo "Cyclomatic-ish score: $N branches"
```

```bash
chmod +x /tmp/kata-05/.claude/skills/review/scripts/complexity.sh
```

Run `/review src/app.js` again. Claude reads only `references/javascript.md` (not python.md), executes `scripts/complexity.sh` directly (no re-derivation), and gives you a better review for fewer tokens.

#### 4. Browse the Official Plugin Marketplace

In a Claude session:

```
/plugin
```

Tab to the **Discover** tab. Browse the official marketplace. Find one that looks useful for your stack and install it. Recommended for a quick win:

- `feature-dev@claude-plugins-official` — opinionated feature workflow
- `code-review@claude-plugins-official` — structured reviewer
- (or browse what fits)

After install, exit the plugin browser and try the new slash command(s) it added.

#### 5. (Stretch) Stand Up a Tiny Private Marketplace

```bash
mkdir -p /tmp/kata-05-mp/.claude-plugin
cd /tmp/kata-05-mp
git init
```

`.claude-plugin/marketplace.json`:

```json
{
  "name": "kata-mp",
  "version": "0.1.0",
  "plugins": [
    {
      "name": "kata-review",
      "description": "Code review skill from kata 05",
      "source": "./plugins/kata-review"
    }
  ]
}
```

Copy your skill into the marketplace:

```bash
mkdir -p plugins/kata-review/skills/review
cp -r /tmp/kata-05/.claude/skills/review/* plugins/kata-review/skills/review/
git add . && git commit -m "v0.1.0"
```

In a fresh project, add the marketplace by local path and install:

```bash
mkdir -p /tmp/kata-05-consumer && cd /tmp/kata-05-consumer && claude
```

Inside the Claude session:

```
/plugin marketplace add /tmp/kata-05-mp
/plugin install kata-review@kata-mp
/review some-file.js
```

You've just shipped a private skill the same way you'd ship one to your whole org via git.

### Discussion Points

- Which of your team's repetitive prompts would benefit from a skill?
- When does progressive disclosure (folder skill) win vs. one big SKILL.md? Anything <200 lines probably doesn't need it.
- Would your team build a private marketplace, or contribute to an existing community one?
- What's the right governance model for who can publish to your internal marketplace?
