```
                                                        \ | /
                                                       --\*/--
                          ___                            / | \
                         /   \
                        | ^ ^ |
                         \_u_/
                      _____|_____
          .---.     /     |     \     .---.
         | O  |---/   /---|---\  \---| O  |
         | /\ |  |   /    |    \  |  | /\ |
         |/  \|  |  |     |     | |  |/  \|
         '----'  |  |     |     | |  '----'
                  \  \    |    /  /
                   \  ----|---  /
                    |    / \   |    how Claude perceives me
                    |   /   \  |    Authored by Claude
                    |  /     \ |
                    | /       \|
                    |/         '
                    '
```

# Protodrake — Claude Code Day 2 Workshop (2026 Edition)

A hands-on workshop for developers learning Claude Code's most powerful features. 11 katas total, ~3 hours if you pick 4–5 of them. Each kata has a quick **Fundamentals** section for newcomers and a meatier **Novel Layer (2026)** section that surprises even daily users.

## Who Is This For?

Two audiences side by side:
- **Newcomers** — have tried Claude Code once or twice, want the foundation
- **Daily users** — want to see what's changed in the 2026 ecosystem (plugins, 27-event hooks, per-agent model pinning, elicitation, statusline, headless workflows)

Both should leave with new tricks.

## Workshop Schedule (~3 hr)

Facilitators pick 4–5 katas based on audience interest. Each takes 20–35 min including a brief demo.

| # | Kata | Time | Fundamentals | Novel layer (2026) |
|---|------|------|--------------|--------------------|
| 0 | [Prompting Patterns](day2-katas/0-prompting-patterns/) | Reference | Universal patterns across Claude Code, Claude.ai, Cursor, Copilot | Spec-driven dev, effort levels, prompt-as-judge, backpressure |
| 0 | [Directory & Config Files](day2-katas/0-directory-and-config-files-example/) | Reference | `CLAUDE.md`, `.claude/`, settings layers | Statusline, agents/, plugins/, env-based model routing |
| 00 | [CLAUDE.md & Sub-Agents Example](day2-katas/00-CLAUDE.mds-subagents-example/) | Reference | Layered `CLAUDE.md`, sub-agent dispatch patterns | YAML-frontmattered agents, per-agent model pinning |
| 01 | [Agent Modes](day2-katas/01-agent-modes/) | ~20 min | Permission modes, Shift+Tab, Plan mode | `/effort`, background mode (`&`), Agent View, `claude -p` |
| 02 | [Security & Containment](day2-katas/02-security-containment/) | ~25 min | Permission rules, `/sandbox`, file boundaries | `.claudeignore`, prompt-type hook as AI judge, HTTP hooks |
| 03 | [Variables & Statusline](day2-katas/03-variables/) | ~25 min | Settings layers, `CLAUDE.md`, env vars | Custom statusline (context %, cost, model), `CLAUDE_CODE_SUBAGENT_MODEL` routing |
| 04 | [Hotkeys & Slash Commands](day2-katas/04-hotkeys/) | ~20 min | Default shortcuts, Vim mode, multiline | `/effort`, `/agents`, `/plugin`, `/handoff`, custom keybindings |
| 05 | [Skills & Plugin Marketplaces](day2-katas/05-skills/) | ~30 min | `SKILL.md`, `/command`, dynamic `!` injection | Progressive disclosure (refs/scripts/examples), `/plugin install`, building a private marketplace |
| 06 | [Hooks Mastery](day2-katas/06-hooks/) | ~30 min | PreToolUse/PostToolUse, exit codes, matchers | 27+ events (UserPromptSubmit, SessionStart, PreCompact), 4 handler types (command/http/prompt/agent), async hooks |
| 07 | [Custom MCP Server with Elicitation](day2-katas/07-custom-mcp-server/) | ~35 min | stdio server, tools/resources, registration | **Elicitation** (server asks user mid-task via form), packaging MCP as a plugin |
| 08 | [Expert Topics](day2-katas/08-expert-topics/) | Reference | (none — pure reference) | Agent Teams, Agent View, Remote Control, headless CI patterns, token economics |

## Suggested 4-Kata Tracks

**"Showing off what 2026 brought"** — for daily users who want the wow:
> 03 (Statusline) → 05 (Skills + Plugins) → 06 (Hooks) → 07 (MCP + Elicitation)

**"Practical foundations"** — for mixed audience where some are new:
> 01 (Agent Modes) → 03 (Variables + Statusline) → 05 (Skills + Plugins) → 06 (Hooks)

**"Enterprise/security focus"** — for regulated environments:
> 02 (Security) → 03 (Variables) → 06 (Hooks) → 08 (Expert: governance, headless)

## Prerequisites

- Claude Code installed and authenticated (`claude` command works). Latest version recommended (`npm install -g @anthropic-ai/claude-code`).
- Node.js 18+ (for kata 07)
- Git
- `jq` available on the PATH (used by hook scripts in 02, 03, 06)
- A terminal (macOS, Linux, or Windows via PowerShell / Git Bash)

## Cross-Platform Support

All katas include setup instructions for macOS/Linux (bash) and Windows (PowerShell). On Windows, statusline and hook scripts run through Git Bash if installed (use forward slashes in paths) or PowerShell directly.

## How to Use

1. Clone this repo
2. Facilitator picks 4–5 katas to cover live
3. Self-paced learners work through the rest after the workshop

Each kata has:
- **Theory — Fundamentals** (2–3 min read, skip if you already know it)
- **Theory — Novel Layer (2026)** (3–5 min read, this is where the value is)
- **Exercise — Fundamentals** (5–10 min hands-on)
- **Exercise — Novel Layer** (10–20 min hands-on)
- **Discussion Points** — team conversation starters

## Running the Workshop

**As a facilitator:**
- Demo each kata briefly before participants start (~3 min)
- Skip "Fundamentals" exercises for experienced groups; do both for mixed
- Use discussion points for group debriefs between katas
- Pair amateur + experienced devs — they teach each other
- Track-B for fast finishers: send them to kata 08 (Expert Topics) for self-study

**Self-paced:**
- Each kata is self-contained with its own setup/teardown
- Recommended order: 01 → 03 → 05 → 06 → 07 → 08 if working through everything
