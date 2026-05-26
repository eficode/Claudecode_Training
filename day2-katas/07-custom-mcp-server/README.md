# Kata 07: Building a Custom MCP Server with Elicitation (~25–35 min)

## Theory — Fundamentals

The **Model Context Protocol (MCP)** is an open standard that lets AI tools connect to external data sources and services through a unified interface. Build a server once; any MCP-compatible client (Claude Code, Claude Desktop, Cursor, etc.) can use it.

### How MCP Works

```
Claude Code (client)  <--stdio/HTTP-->  MCP Server  <-->  Your Data/API
```

An MCP server exposes three primitives:

| Primitive | Purpose | Example |
|-----------|---------|---------|
| **Tools** | Actions Claude can call | `create_ticket`, `query_db`, `deploy` |
| **Resources** | Read-only data endpoints | `file://config`, `db://users` |
| **Prompts** | Reusable prompt templates | `summarize-pr`, `review-code` |

### Configuring MCP Servers

MCP servers are **not** configured in `.claude/settings.json`. They live in their own config or are added via the CLI:

| Scope     | File                  | Shared via git? |
|-----------|-----------------------|-----------------|
| Project   | `.mcp.json` (project root) | Yes |
| User/Local| `~/.claude.json`      | No |

Add via CLI:

```bash
claude mcp add --transport stdio --scope project my-server -- node ./mcp-server/index.js
```

Or write `.mcp.json` directly:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp-server/index.js"]
    }
  }
}
```

### Transport Options

| Transport | Use case |
|-----------|----------|
| **stdio** | Local servers, runs as subprocess |
| **SSE** | Remote servers, HTTP-based streaming |
| **Streamable HTTP** | Newer (2026), recommended for remote |

---

## Theory — Novel Layer (2026)

### Elicitation — The Server Asks the User Mid-Task

Released in Claude Code 2.1.76 (March 14, 2026). An MCP server can pause execution and request structured input from the user via an interactive dialog, then continue with that input. Tools no longer need to anticipate every parameter upfront.

```
Claude Code                    MCP Server
     |---- tools/call --------->|
     |                          |
     |                          |  (needs user decision)
     |<-- elicitation/request --|
     |  [dialog appears in CLI] |
     |---- elicitation/response>|
     |                          |  (continues execution)
     |<--- final result --------|
```

Two modes:
- **Form mode**: server provides a JSON Schema; Claude Code renders form fields
- **URL mode**: server provides a URL; user completes flow in browser, then confirms in CLI

Real use cases:
- Deploy tool: "Which env? Confirm prod?"
- Email tool: "Pick a template from this list"
- Migration tool: "Validate the transformed sample before applying"

To auto-respond (for headless/CI), use the new `Elicitation` and `ElicitationResult` hooks.

### Packaging an MCP Server as a Plugin

You don't have to ship MCP servers via `.mcp.json` anymore. Bundle into a plugin and distribute through a marketplace (kata 05). The plugin manifest lists the MCP server, and `/plugin install` wires up `.mcp.json` automatically. One install, one source of truth.

---

## Exercise

### Setup

**macOS / Linux:**

```bash
mkdir -p /tmp/kata-07/mcp-server && cd /tmp/kata-07
git init
npm init -y
npm install @modelcontextprotocol/sdk zod
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path $env:TEMP\kata-07\mcp-server | Out-Null
Set-Location $env:TEMP\kata-07
git init
npm init -y
npm install @modelcontextprotocol/sdk zod
```

Add `"type": "module"` to `package.json`.

### Tasks — Fundamentals

#### 1. Build a Simple Tool Server

`mcp-server/index.js`:

```javascript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "workshop-tools", version: "1.0.0" });

// Generate a UUID
server.tool("generate_uuid", "Generate a random UUID v4", {}, async () => ({
  content: [{ type: "text", text: crypto.randomUUID() }],
}));

// Count words
server.tool(
  "count_words",
  "Count words in a given text",
  { text: z.string().describe("The text to count words in") },
  async ({ text }) => ({
    content: [{ type: "text", text: `Word count: ${text.trim().split(/\s+/).filter(Boolean).length}` }],
  })
);

// Format JSON
server.tool(
  "format_json",
  "Pretty-print a JSON string",
  { json: z.string().describe("JSON string to format") },
  async ({ json }) => {
    try {
      return { content: [{ type: "text", text: JSON.stringify(JSON.parse(json), null, 2) }] };
    } catch (e) {
      return { content: [{ type: "text", text: `Invalid JSON: ${e.message}` }], isError: true };
    }
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

#### 2. Register and Test

```bash
claude mcp add --transport stdio --scope project workshop-tools -- node mcp-server/index.js
```

Or write `.mcp.json` directly:

```json
{
  "mcpServers": {
    "workshop-tools": {
      "type": "stdio",
      "command": "node",
      "args": ["mcp-server/index.js"]
    }
  }
}
```

Start Claude and try:
1. `"Generate a UUID"`
2. `"Count the words in: The quick brown fox jumps over the lazy dog"`
3. `"Format this JSON: {\"name\":\"test\",\"value\":42}"`
4. `/mcp` — see connected servers and tools

### Tasks — Novel Layer

#### 3. Add a Tool With Elicitation

Add to `mcp-server/index.js`:

```javascript
// Deploy tool — uses elicitation to ask for environment confirmation mid-task
server.tool(
  "deploy",
  "Deploy the current branch. Asks the user which environment via an interactive dialog.",
  { service: z.string().describe("Service name to deploy") },
  async ({ service }, { sendNotification, server: srv }) => {
    // Request structured input from the user
    const response = await srv.elicitInput({
      message: `You're about to deploy '${service}'. Confirm the target environment.`,
      requestedSchema: {
        type: "object",
        required: ["env", "confirmed"],
        properties: {
          env: {
            type: "string",
            enum: ["dev", "staging", "production"],
            description: "Target environment",
          },
          confirmed: {
            type: "boolean",
            description: "I have verified the deploy is safe",
          },
          notes: {
            type: "string",
            description: "Optional deploy notes (visible in audit log)",
          },
        },
      },
    });

    if (response.action !== "accept") {
      return { content: [{ type: "text", text: `Deploy cancelled by user.` }] };
    }

    const { env, confirmed, notes } = response.content;
    if (!confirmed) {
      return { content: [{ type: "text", text: `Deploy aborted: user did not confirm.` }] };
    }

    // In real life: trigger the deploy here. We just log it.
    return {
      content: [{
        type: "text",
        text: `✅ Deployed '${service}' to ${env}.${notes ? ` Notes: ${notes}` : ""}`,
      }],
    };
  }
);
```

Restart Claude and ask: `"Deploy the user-service"`.

You'll see Claude call the `deploy` tool, then a form dialog opens in your terminal asking which environment and for confirmation. Fill it in and watch the tool resume with your answers.

This pattern is the missing piece for high-stakes actions that need human confirmation **without** the tool author predicting every possible parameter combination upfront.

#### 4. Add a Resource (Bonus)

```javascript
import fs from "fs";

server.resource("project-info", "project://info", async (uri) => {
  const pkg = JSON.parse(fs.readFileSync("./package.json", "utf-8"));
  return {
    contents: [{ uri: uri.href, mimeType: "application/json", text: JSON.stringify(pkg, null, 2) }],
  };
});
```

Ask Claude: `"Fetch the project://info resource and tell me what version we're on."`

#### 5. (Stretch) Package as a Plugin

Building on kata 05's marketplace pattern, you can ship this MCP server through `/plugin install`:

```
your-marketplace/
└── plugins/
    └── workshop-tools/
        ├── .claude-plugin/plugin.json         # references the MCP entry below
        ├── mcp.json                           # this plugin's MCP server config
        └── server/index.js                    # the server itself
```

`mcp.json` inside the plugin:

```json
{
  "mcpServers": {
    "workshop-tools": {
      "type": "stdio",
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server/index.js"]
    }
  }
}
```

Now `/plugin install workshop-tools@yourmp` is one command instead of "clone this repo, install deps, edit .mcp.json, restart."

### Discussion Points

- What internal APIs/databases at your company would benefit from an MCP server?
- Where would **elicitation** make sense? Deploy? DB migrations? Spending money?
- stdio vs. streamable HTTP — when does each win?
- How would you handle auth for an MCP server that accesses sensitive data?
- Skills vs MCP tools — when is each the right tool? (Hint: skills shape *how* Claude works; MCP servers add *what* Claude can reach.)
