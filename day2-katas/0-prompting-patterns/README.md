# Prompting Patterns for AI-Assisted Development

Universal patterns that work across Claude Code, Claude.ai, Cursor, and Copilot — plus 2026-era patterns that only make sense once you're treating Claude as an agent, not an autocomplete.

---

## Core Patterns (work everywhere)

### 1. Be Specific About Output

"Make this better" gives you a coin flip. "Refactor this function to use async/await and remove the callback parameter" gives you exactly what you need. State the transformation, not the vibe. Include the format if it matters: "return a markdown table," "use bullet points," "just the shell command, no explanation."

### 2. Front-Load Context and Constraints

Put boundaries before the ask. Claude reads top-down and anchors on early instructions.

**Weak:** "Add pagination to the users endpoint. Oh and only use existing utils."
**Strong:** "Using only the existing utils in `src/lib/`, add cursor-based pagination to the users endpoint."

### 3. Show the Shape You Want

A 3-line example of desired output beats a paragraph of description. Models are excellent at pattern-matching from examples.

### 4. Decompose, Don't Monologue

One clear task per message outperforms a wall of requirements. A 15-step prompt might get steps 1-8 right and hallucinate the rest. Five 3-step prompts rarely go sideways.

### 5. Constrain Scope Explicitly

AI models are eager — they'll "improve" surrounding code, add error handling you didn't ask for, and refactor imports while fixing a typo. Set explicit boundaries:

- "Only modify files in `src/auth/`"
- "Don't change the tests"
- "Fix the null check on line 42 — nothing else"

### 6. Ask for Plans Before Code

"How would you approach this?" is the highest-leverage prompt in your toolkit. It surfaces misunderstandings before any code is written. Especially valuable for multi-file changes or unfamiliar codebases.

### 7. Correct by Contrast

"Don't use class inheritance, use composition instead" is clearer than just "use composition." The contrast gives the model a negative example and a positive one.

### 8. Iterate, Don't Restart

"Now add error handling to what you just wrote" preserves context. Starting a new conversation throws away everything the model learned about your intent.

### 9. Use Persistent Instructions for Repeated Conventions

If you find yourself repeating instructions ("always use bun not npm," "we use tabs"), move them into persistent config (`CLAUDE.md`, `.cursorrules`, GitHub Copilot instructions).

### 10. Name the Task Type

"This is a bug fix" or "this is a refactor" primes the model differently. A bug fix should be surgical. A refactor should preserve behavior. A new feature needs more creativity.

---

## 2026 Patterns (agent-era, not autocomplete-era)

These patterns only make sense once your tool can read your whole repo, run commands, and spawn sub-instances of itself.

### A. Spec-Before-Code (Interview-to-Spec)

For anything bigger than a single function, run a short interview first. Ask Claude to interrogate *you* before it codes:

> "I want to add a notification system. Before you propose anything, ask me 5 questions you'd need answered to design this well. Don't suggest yet, just ask."

Then turn the answers into a written spec (saved to `docs/specs/`), and only after that ask Claude to implement against the spec. Sessions stop falling apart at hour three.

### B. Delegate Heavy Reads to Subagents

Don't burn your main context on a 5,000-line file. Tell Claude to spawn an Explore subagent:

> "Use a subagent to read `src/legacy/billing.ts` and return only: the public API surface, the data flow, and any obvious bugs. Don't return code."

The subagent's context dies when it returns; your main session stays lean.

### C. Match the Effort Level to the Task

Claude Code exposes effort levels (`/effort low|medium|high|max|auto`). Trivial edits → low. Architecture decisions → high or max. Don't run max on everything — you pay for thinking tokens you don't need.

### D. The "Backpressure" Loop

Give the agent something to react to: a test command, a linter, a type checker. "Run `pnpm typecheck` after each edit and fix anything red." Backpressure from a tool is more reliable than prose like "be careful with types."

### E. Prompt-as-Judge for Quality Gates

Wrap a high-stakes step in an automatic review pass:

> "After implementing, run a second pass: act as a code reviewer and find three issues with what you just wrote. Then fix them."

Same model, fresh context, separate pass. Catches roughly what a human reviewer would catch.

### F. Use Plan Mode Aggressively

Plan mode (Shift+Tab to cycle) is read-only. Use it for any change you're not 100% sure about — let Claude produce a plan, you approve, *then* execute. Costs a single round trip and prevents whole categories of "Claude did something I didn't want."

---

## Prompting by Tool: Use Cases and Examples

Each tool has a different interaction model, context window, and set of capabilities. The same intent often needs a different prompt shape.

### Claude Code (CLI / agentic)

Full filesystem access, runs commands, reads `CLAUDE.md`. Be ambitious; let it explore.

| Use Case | Prompt Example |
|----------|---------------|
| **Bug fix** | `"The /api/users endpoint returns 500 when email is null. Find the handler, fix the null check, and add a unit test."` |
| **Feature** | `"Add rate limiting to all POST routes. Use the existing redis client in src/cache/client.ts. 100 requests per minute per IP."` |
| **Refactor** | `"Move all raw SQL from src/services/ into src/db/queries/. Update imports. Don't change behavior."` |
| **Explore** | `"How does the auth flow work? Trace from login route to session creation. Use a subagent."` |
| **Plan first** | `"Enter plan mode. Propose an approach for adding WebSocket support for live notifications."` |
| **Parallel work** | `"Spawn 3 subagents: one finds all auth-related files, one finds session code, one finds middleware. Synthesize."` |

### Claude.ai (chat)

No file access, no execution. You bring the code as pasted snippets.

| Use Case | Prompt Example |
|----------|---------------|
| **Debug a snippet** | `"This function throws on line 12. Here's the function: [paste]. What's wrong?"` |
| **Design / architecture** | `"I'm building a notification system. Suggest a pattern that's extensible to new channels — no code yet."` |
| **Code review** | `"Review this diff for SQL injection and auth bypass: [paste]"` |
| **Explain** | `"Explain what this regex does step by step: /^(?=.*[A-Z])(?=.*\d)[A-Za-z\d@$!%*?&]{8,}$/"` |
| **Compare approaches** | `"Tradeoffs between cursor-based and offset pagination for a REST API with ~1M rows?"` |

### Cursor (IDE-integrated)

Sees open files. Use `@` references heavily (`@file`, `@codebase`, `@docs`, `@web`).

| Use Case | Prompt Example |
|----------|---------------|
| **Inline edit (Cmd+K)** | Select function, then: `"Convert to async/await"` |
| **Chat with codebase** | `"@codebase How is authentication handled?"` |
| **Composer (agent)** | `"Add a dark mode toggle. Use the existing theme context in @file:src/contexts/theme.tsx."` |

### GitHub Copilot

Primarily autocomplete with a chat sidebar. Lead by example — write one instance of the pattern, it replicates.

| Use Case | Prompt Example |
|----------|---------------|
| **Comment-driven** | `// Parse CSV, skip header, return array of {name, email, role}` — let Copilot fill in |
| **Chat** | `"Write a unit test for #file:utils.ts #selection"` |
| **Fix** | `"/fix"` with an error selected |

---

## Quick Comparison

| Capability | Claude Code | Claude.ai | Cursor | Copilot |
|-----------|-------------|-----------|--------|---------|
| File system access | Full | None | Full (workspace) | Limited (open files) |
| Command execution | Yes | No | Yes (agent mode) | No |
| Multi-file edits | Native | Manual paste | Native | Limited |
| Persistent config | `CLAUDE.md`, `.claude/` | Projects | `.cursorrules` | Instructions file |
| Subagents / parallel | Native | No | Limited | No |
| Best for | Multi-step, refactors, features | Design, review, explanation | In-IDE edits, codebase Q&A | Autocomplete, boilerplate |
| Prompt style | Outcome-oriented, ambitious | Self-contained, context-rich | Reference-heavy (`@`), surgical | Implicit, comment-driven |
