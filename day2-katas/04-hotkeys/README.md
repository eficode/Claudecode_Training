# Kata 04: Hotkeys, Slash Commands & Shortcuts (~15–25 min)

## Theory — Fundamentals

Claude Code has built-in keyboard shortcuts and supports full customization via `~/.claude/keybindings.json`.

### Essential Default Shortcuts

| Shortcut | Action |
|----------|--------|
| `Enter` | Submit message |
| `Ctrl+C` | Cancel current operation (hardcoded) |
| `Ctrl+D` | Exit Claude Code (hardcoded) |
| `Ctrl+L` | Clear screen |
| `Shift+Tab` | Cycle permission modes |
| `Esc Esc` | Rewind/undo last response |
| `Ctrl+R` | Search command history |
| `Ctrl+G` | Open prompt in external editor (`$EDITOR`) |

### Multiline Input

| Terminal | Shortcut |
|----------|----------|
| Any | `\` then `Enter` |
| macOS Terminal | `Option+Enter` |
| iTerm2, WezTerm, Ghostty, Kitty | `Shift+Enter` |
| Windows Terminal, PowerShell | `Shift+Enter` |

### Vim Mode

Enable via `/config` → **Editor mode** → `vim`, or set in settings:

```json
{ "editorMode": "vim" }
```

Provides standard vi NORMAL/VISUAL editing: `Esc` for normal, `i`/`a`/`o` for insert, `h/j/k/l` for nav, `d`/`c`/`y`/`p` for delete/change/yank/paste. Use `o`/`O` or `Ctrl+J` for newlines — `Enter` still submits.

---

## Theory — Novel Layer (2026)

### The Slash Commands That Actually Matter Day-to-Day

The slash-command surface has grown a lot. The ones you'll use most:

| Command | What it does |
|---------|-------------|
| `/effort low\|medium\|high\|xhigh\|auto` | Set thinking budget for next turns |
| `/agents` | Interactive subagent generator (writes YAML for you) |
| `/plugin` | Plugin browser, search, install, enable/disable |
| `/handoff` | Compact-and-restart with a summary, when context is full |
| `/context` | Show current context usage breakdown by source |
| `/resume` | Pick a past session to resume |
| `/worktree` | Create or switch to a git worktree (parallel branches) |
| `/keybindings` | Interactive keybindings editor |
| `/permissions` | Interactive permission manager |
| `/hooks` | Hook config viewer (read-only since March 2026) |
| `/mcp` | List connected MCP servers + tools |
| `/sandbox` | Toggle OS-level sandbox |
| `/config` | Settings UI |
| `/init` | Generate a CLAUDE.md for the current project |

Press `/` and pause — you get autocomplete with descriptions for every available command (including any installed via plugins or skills).

### `/handoff` — The Trick That Saves Long Sessions

When you hit ~70% context, output quality starts degrading (context rot). `/handoff` summarizes the current session into a compact resumable note and starts a fresh session — same model, same project, lean context.

A discipline that works: watch the statusline (kata 03). At 60%, ask Claude to update `CLAUDE.md` with anything you'll want next session. At 75%, `/handoff`.

### Custom Keybindings

File: `~/.claude/keybindings.json` (macOS/Linux) or `%USERPROFILE%\.claude\keybindings.json` (Windows)

```json
{
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+e": "chat:externalEditor",
        "meta+return": "chat:submit",
        "ctrl+h": "chat:handoff"
      }
    }
  ]
}
```

Available contexts: `Global`, `Chat`, `Autocomplete`, `Confirmation`, `Transcript`, `Help`, more.

---

## Exercise

### Tasks — Fundamentals

#### 1. Learn the Essentials

```bash
cd /tmp && claude
```

Practice:

1. Type a message, then `Escape` — message clears
2. Submit with `Enter`
3. `Ctrl+L` — clear screen
4. `Shift+Tab` — cycle permission modes (watch the indicator)
5. `Ctrl+R` — search command history
6. `Esc Esc` — rewind last response

#### 2. Multiline Input

In the same session:
1. Type `Tell me about` then `\` and `Enter` — continue on next line
2. Or `Option+Enter` (macOS) / `Shift+Enter` (iTerm2, Windows Terminal)
3. Write a 3-line prompt and submit

#### 3. Try Vim Mode

`/config` → Editor mode → `vim`. Practice `Esc` → `h/j/k/l` → `i`. Submit with `Enter` (use `o`/`Ctrl+J` for newlines). Switch back to `normal` when done.

### Tasks — Novel Layer

#### 4. The Effort Dial

```
/effort low
What's 2+2?
```

```
/effort high
Design a rate limiter that handles burst traffic without starving long-running requests. Plan only.
```

Notice the response-time and thinking-depth difference. Now `/effort auto` to let Claude pick per turn.

#### 5. Browse Slash Commands

Press `/` and pause. Read through the autocomplete list. Try:

- `/context` — see how your context is being spent right now
- `/mcp` — see any connected MCP servers
- `/plugin` — open the plugin browser (Discover tab)

#### 6. Try `/handoff`

Have a long-ish chat that fills up some context (ask a few exploratory questions about a real project). Then:

```
/handoff
```

You get a fresh session with a summary loaded in. Compare your statusline context % before and after.

#### 7. Custom Keybinding

Create `~/.claude/keybindings.json`:

```json
{
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+e": "chat:externalEditor"
      }
    }
  ]
}
```

In Claude, press `Ctrl+E` — your `$EDITOR` opens for composing a longer prompt. Save and quit and the buffer is your message.

#### 8. Explore the Keybindings Editor

In Claude, run `/keybindings` to browse all available actions and contexts. Pick one or two more to bind.

### Discussion Points

- Which shortcuts will you use daily? Weekly? Never?
- Where does `/handoff` fit in your workflow? Would you script it from a hook?
- Would you bind `chat:handoff` to a key? Why or why not?
- Vim mode vs. external editor (`Ctrl+G`) — which fits your habits?
